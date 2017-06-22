---
title: 实现一个简单的Promise队列
desc: Promise本身的特性就是“保证在未来某一时刻会完成”，因此可以透明地加入队列，只要在未来某一时刻取出并处理，就可以继续向后执行原有代码，不会改变原有结构。  
date: 2017-6-21 22:00:00.000  
tags: 
- javascript
- Promise
- ES6
author: ngtmuzi  
category: 神秘代码  
---

*原博客创建于 2016-9-18 22:34:48， 更新于 2017-6-21 22:00:00*

## 需求

来自一个很现实的需求，有个业务只能串行执行，原来的代码的逻辑是弄个全局锁变量，当新的请求来的时候检查变量，在锁时直接返回错误让对方做重试，非常暴力。

很直观的改进方案就是用队列，系统是个用redis实现的伪消息队列，来多少推多少，根本做不到控流，而在代码内加队列又要考虑不能对原来的代码有太大改动（150行严重缩进横跨3个系统的业务代码，怼不动），还好原代码就是Promise写的，做流程控制好歹比callback简单多了

## 思路

* `Promise`本身的特性就是“保证在未来某一时刻会完成”，因此可以透明地加入队列，只要在未来某一时刻取出并处理，就可以继续向后执行原有代码，不会改变原有结构。

* 队列本身要有节流功能，即可以控制同一时间内在运行的`Promise`数量，参考`bluebird`的`map`函数中的concurrency(并发)字段。

## 实现

比较核心的代码简化起来就这一段：
```javascript
  function add(fn) {
    return new Promise((resolve, reject) => {
      this.queue.push(() =>
        Promise.resolve()
          .then(fn)
          .then(resolve, reject)
      );
    });
  }
```

可以看到我们是往`queue`队列里加入了一个函数，这个函数包裹了原函数`fn`，将它执行的同步或者异步的结果传给外层的`Promise`，这样对外表现就还是一个`Promise`，这个函数进入队列，等待轮到它执行的时机

之后就是在内部维护一个“正在运行的任务数量”，在`fn`运行前后做加减和判断，就可以控制并行数了
  
完整代码：[np-queue](https://github.com/ngtmuzi/np-queue)  

运行起来的感觉类似这样

```javascript
const q = new Queue();
const delay = (value) =>  
  new Promise(resolve => {
    setTimeout(() => resolve(value), 1000);  
  });

q.add(()=>delay(1)).then(console.log);
q.add(()=>delay(2)).then(console.log);

const delay_wrap = q.wrap(delay);

delay_wrap(3).then(console.log);
delay_wrap(4).then(console.log);
```

默认并发数是1，因此代码会相隔1秒依次输出`1,2,3,4`

放到业务代码上，原代码是`delay().then(...)`，现在改为`q.add(delay).then(...)`，业务逻辑仍旧能跟在后面，原代码改动比较小，也易于理解

再配合`wrap()`方法我们能更透明地完成这件事，它直接给一个函数提供了一层包装：
```javascript
const wrapFn = function () {
  return queue.add(fn.bind(thisArg, ...arguments));
};
```
原代码为`delay(a,b).then(...)`我们可以改为`delay_wrap(a,b).then(...)`，参数都不用改，透明地加了一层控制并发的逻辑，感觉很好。