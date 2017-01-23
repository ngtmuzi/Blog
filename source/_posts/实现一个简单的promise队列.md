---
title: 实现一个简单的promise队列
desc: promise本身的特性就是“保证在未来某一时刻会完成”，因此可以透明地加入队列，只要在未来某一时刻取出并处理，就可以继续向后执行原有代码，不会改变原有结构。  
date: 2016-9-18 22:34:48.448
tags: 
- javascript
- Promise
- ES6
author: ngtmuzi  
category: 神秘代码  
---
　　来自一个很现实的需求，有个业务只能串行执行，原来的代码的逻辑是弄个全局锁变量，当新的请求来的时候检查变量，在锁时直接返回错误让对方做重试，非常暴力。
出于对接方的强烈要求要改，很直观的改进方案就是用队列，原来的系统是个用redis实现的伪消息队列，来多少推多少，根本做不到控流，而在代码内加队列又要考虑不能对原来的代码有太大改动（150行严重缩进横跨3个系统的业务代码，怼不动），还好原代码就是`promise`写的，做流程控制好歹比callback简单多了，于是思路如下：

* `promise`本身的特性就是“保证在未来某一时刻会完成”，因此可以透明地加入队列，只要在未来某一时刻取出并处理，就可以继续向后执行原有代码，不会改变原有结构。

* 队列本身要有节流功能，即可以控制同一时间内在运行的`promise`数量，参考`bluebird`的`map`函数中的concurrency(并发)字段。

　　其实大体思路就是模仿bluebird中的map函数，不同点在于map是处理一个固定的数组，且会结束（当然可以动态地修改该数组，但不推荐），而队列会一直待机处理（伪，实际是队列有改动时才做检查）  
于是完成代码：[promiseQueue.js](https://github.com/ngtmuzi/wheel/blob/master/tools/promiseQueue.js)
运行起来的感觉类似这样
```javascript
const a = Queue(2);
const delay = () => Promise.delay(1000,new Date());

a.add(delay).then(console.log);
a.add(delay).then(console.log);
a.add(delay).then(console.log);
```
　　并发数是2，因此第3个promise会在前面1个执行完后才开始执行，运行结果
```
//2016-09-18T14:29:26.490Z
//2016-09-18T14:29:26.491Z
//2016-09-18T14:29:27.495Z
```
　　这个轮子的重点是，原代码是`delay().then(...)`，现在改为`a.add(delay).then(...)`，业务逻辑仍旧能跟在后面，原代码改动比较小，也易于理解