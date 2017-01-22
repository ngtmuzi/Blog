---
title: 自己撸一个Promise库的过程
desc: 相信学过nodejs的人都必然会接触到nodejs中的流（stream） 
date: 2017-1-19 9:54:47.349
tags: 
- Promise
- ES6  
author: ngtmuzi  
category: 神秘代码
---
新年第一篇博客，写了一年多的JS，感觉只是弄懂了皮毛，没有好好的钻研过，关于前端那方面更是完全空白，看了别人的面试经历后深感忧虑。正好年前比较闲，重构的项目也基本稳定了，于是就无聊想研究下代码，`Promise`从ES6出来就用到现在，`then/catch`那一套倒是很熟了，但还是不清楚里面的原理，就趁此机会深入理解一下

## 参考资料：

>[Promises/A+规范](https://promisesaplus.com/)  
>[崔鹏飞的博客](http://cuipengfei.me/blog/2016/05/15/promise/)  
>[Promise测试库](https://github.com/promises-aplus/promises-tests)

从我的结果来说，按照规范上的规则一行一行地写，配合测试库一遍一遍地跑，基本上都能完成代码，而且还有一种类似闯关的感觉（参见提交记录），挺爽，可以试试。

## 注意点
因为规范说得实在太够详细了，基本也没什么好介绍的了，说下一些需要注意的点吧：

* 三个状态，运行中(pending)，已完成(resolved)，已失败(rejected)
* **规范2.2.4**：`onFulfilled`或`onRejected`在运行至上下文堆栈仅剩平台代码前不能运行；说得很绕，主要是指这2个函数需要异步执行（即使`Promise`已经`resolved`），而常用的异步执行函数就是`setTimeout`、`setImmediate`以及node特有的`process.nextTick`等函数，具体函数的不同会影响到整个`Promise`执行的效率，这里可以关注一下
* **规范2.2.6.1**：当`Promise`进入完成状态，所有`onFulfilled`都需要按它们调用`then`的顺序来触发；这里隐含了一个点：`then`里传进去的函数是通过主动回调来触发的，也就是说`Promise`本质上是回调的封装，只是对状态和规范做了严格限制，使得最后使用的时候方便不少
* **规范2.3**：Promise解决程序，规范里将它表示为`[[Resolve]](promise, x)`，实际上就是写一个函数，输入一个未完成的`promise`和值`x`，通过一系列规则判断，以确定这个`promise`最后的状态。这部分就是Promise规范的核心了，因为规则写得十分详细，照着来写基本都可以过，关注点：要保证自己传进`x.then`的函数仅能被调用一次，全程记得用`try-catch`包裹

这样基本就差不多了，从零写出来之后对Promise理解还是增加了不少的，之后可以尝试加入一些`bluebird`的常用方法，比如`try`、`all`、`map`之类的，对理解代码逻辑很有帮助。

附上[我的Promise库](https://github.com/ngtmuzi/np/blob/master/index.js)。另外要说一句，多尝试`new Promise()`来自己封装异步代码，我是见过不少同事只懂从`Promise.resolve()`开始的，那就弄丢了Promise最强大的部分