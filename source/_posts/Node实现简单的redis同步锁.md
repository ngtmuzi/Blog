---
title: Node实现简单的redis同步锁
desc: 照着作者的思路来实现还是很容易的
author: ngtmuzi
category: 神秘代码
date: 2018-03-22 14:07:34
tags:
- nodejs
- redis
---

也是实际的需求，某个业务有并发问题，同时处理会引起脏读脏写，之前实现了一个[promise队列](https://ngtmuzi.com/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84promise%E9%98%9F%E5%88%97/)就是来解决这件事的，但现在服务器部署了多台，业务方随机访问，没办法在内存层面做到队列控制了，因此就想到用redis来实现一个简单锁来控制并发

## 参考资料
> [Distributed locks with Redis](https://redis.io/topics/distlock)

来自官网上redis作者的文章，虽然是讲分布式锁redLock的，但也提到了使用redis实现简单锁的方法，并提出了他认为的简单锁的缺点：
* 单点故障
* 有部署主从的情况下，可能主服上的锁定操作还没同步到从服，主服就出现了故障，从服晋升为主服，使得之前的锁定不生效

## 分析

在我这边的实际业务上看，redis的故障是可以容忍的，实话说我接触了redis挺长一段时间还从没见它崩过，因此就直接照着作者的思路来实现一个简单锁就好了：

1. 客户端使用`SET NX`语法设置一个会过期的键，当键存在时返回锁定错误（即表明已经这个键已经被别人锁着了）
    ```
    SET resource_name my_random_value NX PX 30000
    ```
1. 解锁时向redis服调用一段`lua`脚本
    ```lua
    if redis.call("get",KEYS[1]) == ARGV[1] then
        return redis.call("del",KEYS[1])
    else
        return 0
    end
    ```
    解锁时传递的value必须与锁定时的value相等，这是用于防止其他客户端在错误情况下会解锁其他人锁的情况，就是“解铃还须系铃人”的那种感觉
1. 若超过过期时间，客户端还没发起解锁，那么该键将会因为过期而被redis删除，避免产生死锁的情况
1. 更完善一点实现还会考虑加时的情况，即延长自己的锁定时间，也需要用lua脚本来做判断value是否相等

## Node代码实现

使用[ioredis模块](https://www.npmjs.com/package/ioredis)

### 构造函数
```javascript
class Locker {
  constructor(redis) {
    this.redis   = redis;
    this.lockMap = new Map();

    //定义lua脚本让它原子化执行
    this.redis.defineCommand('lua_unlock', {
      numberOfKeys: 1,
      lua         : `
        local remote_value = redis.call("get",KEYS[1])
        
        if (not remote_value) then
          return 0
        elseif (remote_value == ARGV[1]) then
          return redis.call("del",KEYS[1])
        else
          return -1
        end`
    });
  }
}
```
传递一个ioredis实例进来，`lockMap`用来在内存在维护多组锁定相关的键值对，使用ioredis的功能定义一个解锁用的lua脚本以待后面调用，脚本稍微增加了一点内容

### 加锁
```javascript
/**
 * 锁定key，如已被锁定会抛错
 * @param key
 * @param expire    过期时间(毫秒)
 * @return {Promise<void>}
 */
async function lock(key, expire = 10000) {
  const value = crypto.randomBytes(16).toString('hex');

  let result = await this.redis.set(key, value, 'NX', 'PX', expire);
  if (result === null) throw new Error('lock error: key already exists');

  this.lockMap.set(key, {value, expire, time: Date.now()});
  return 'OK';
}
```
生成一个随机值做value，写入redis和内存中

### 解锁
```javascript
/**
 * 解锁key，无论key是否存在，解锁是否成功，都不会抛错（除网络原因外），具体返回值:
 * null: key在本地不存在    0:key在redis上不存在    1:解锁成功      -1:value不对应，不能解锁
 * @param key
 * @return {Promise<*>}
 */
async function unLock(key) {
  if (!this.lockMap.has(key)) return null;
  let {value, expire, time} = this.lockMap.get(key);
  this.lockMap.delete(key);

  return await this.redis.lua_unlock(key, value);
}
```
从内存中找到对应key的value，把它们传给redis，使用lua脚本解锁，因为解锁基本算是个收尾的工作，因此各种没解锁成功的情况我不会抛错，有需要可以根据返回值自己处理

### 等待加锁
```javascript
/**
 * 每隔interval时间就尝试一次锁定，当用时超过waitTime就返回失败
 * @param key
 * @param expire
 * @param interval
 * @param waitTime
 * @return {Promise<void>}
 */
async function waitLock(key, expire, interval = 500, waitTime = 5000) {
  let start_time = Date.now();
  let result;
  while ((Date.now() - start_time) < waitTime) {
    result = await this.lock(key, expire).catch(() => {});
    if (result === 'OK') return 'OK';
    else await delay(interval);
  }
  throw new Error('waitLock timeout');
}

/**
 * 等待一段时间（毫秒）
 * @param ms
 */
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

```
对键重复地尝试加锁，直到抢占到锁资源，类似“连接池”的那种感觉

[完整代码](https://github.com/ngtmuzi/wheel/blob/master/services/redisLocker.js)

## 总结  

redisLock的逻辑有点太复杂了，一般业务用简单的同步锁就好了