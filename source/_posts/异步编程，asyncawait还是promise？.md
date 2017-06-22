---
title: 异步编程，async/await还是promise？  
desc: 还是bluebird好用啊  
date: 2017-4-19 19:00:14  
tags: 
- promise
- async
- es7
author: ngtmuzi  
category: 神秘代码

---

Node.js的`v7.6`版本出来之后`async/await`正式成为可用特性，其最大的亮点就是将异步的逻辑写在同步的代码里，还能捕捉到异步错误，成为了异步编程的最佳实践，但我们真的能完全抛弃掉`promise`吗？  

* Node的自带模块提供的异步接口都是回调式的，想封成async函数你起码还得用类似bluebird的`Promise.promisify`来封装一遍
* 同步的错误捕获也并不一定就最好，起码`try/catch`比起`.catch()`还是要多上几行的，想要在各层代码做捕获还要多加几层`try/catch`，看起来很乱
* `async`定义的函数所返回的是javascript原生的`Promise`对象，也就是说第三方Promise库所提供的各种特性不能在`async`函数后直接使用了，外面还要再包一层`Promise.resolve()`才能转成当前所用的Promise对象，考虑到这点挺烦人的，有些小特性我就干脆直接往原生`Promise`的原型链上挂了


异步编程必然要考虑超时，但使用`async`定义的函数并没有这么方便的功能，那我们就参考`bluebird`的格式加到原生`Promise`原型上好了：
```javascript
Promise.prototype.timeout = function (ms) {
  return new Promise((resolve, reject) => {
    this.then(resolve, reject);
    setTimeout(() => reject(new Error('promise timeout')), +ms);
  });
};
```

也可以在`new Promise()`时直接使用第三方的Promise库，这样就相当于调过`.timeout()`的`promise`都做了一层包装了，唯一注意的一点是在这段代码之前可不要用类似`global.Promise = require('bluebird')`的语句替换掉原生`Promise`对象


最后总结下自己对于`async/await`/`Promise`的一些抉择：
* 业务逻辑代码可以多用`async/await`，以便在各种条件分支中同步地调用异步代码和捕捉错误
* 注重流程的，没有太多分支的较底层代码，和已经使用`Promise`开发的旧代码，没有必要转成`async/await`，各种Promise库提供的特性更方便实现各种复杂的异步逻辑


简言之，就是业务逻辑多用`async/await`，底层代码多用`Promise`