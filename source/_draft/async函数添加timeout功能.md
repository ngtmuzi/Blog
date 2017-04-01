---
title: async函数添加timeout功能  
desc: 还是bluebird好用啊  
date: 2017-2-23 12:00:14  
tags: 
- promise  async
author: ngtmuzi  
category: 神秘代码
---
　　Node.js的`v7.6`版本出来之后`async/await`正式成为可用特性，回想起以前只能在`babel`和`co`里尝试这些新特性，真是恍如隔世。`async/await`最大的亮点就是将异步的逻辑写在同步的代码里，还能捕捉到异步错误，成为了异步编程的最佳实践，但我们真的能完全抛弃掉`promise`吗？  
　　首先Node的自带模块提供的异步接口都是回调式的，想封成async函数你起码还得用`Promise.promisify`来封装一遍，其次同步的错误捕获也并不一定就最好，起码最外层加`try/catch`