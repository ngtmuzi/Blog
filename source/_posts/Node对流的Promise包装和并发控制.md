---
title: Node对流的Promise包装和并发控制
desc: 超简单代码
author: ngtmuzi
category: 神秘代码
date: 2018-11-06 20:47:28
tags:
- stream
- nodejs
- promise
---

最近没有在做直接开发的工作，都是一些旧工作的接手和整理脚本流程之类的，发个之前写的函数吧。主要封装了流（或类似流的类，比如node.js自带的`readline`模块）到`Promise`中，并提供并发数控制的机制（当然需要流本身支持pause才行）。我主要用于读数据库或文件之类的操作，将流的细节封起来感觉还是舒服一点。

```javascript
/**
 * 将可读流传给遍历器fn（可异步），使用流的特性做并发控制和收集返回（注意内存消耗）
 * @param readable       {ReadableStream} 可读流
 * @param fn             {Function}       遍历器，触发时机为data事件
 * @param concurrency    {Number}         并发处理的数量，当并发数满时，流将会被自动暂停
 * @param collectResults {Boolean}        是否收集fn执行的结果，并最后返回结果的数组
 * @param eventName      {String}         事件名，一般是data
 * @return {Promise}      fn执行的次数或结果数组
 */
function streamQueue(readable, fn, { concurrency = 1, collectResults = false, eventName = 'data' } = {}) {
  let runCount = 0, index = 0, isOver = false;
  const results = [];
  return new Promise((resolve, reject) => {
    const checkFinish = () => {
      if (isOver && runCount === 0) resolve(collectResults ? results : index);
    };

    readable.on(eventName, async data => {
      let myIdx = index++;
      runCount++;
      if (runCount >= concurrency) readable.pause();
      try {
        let result = await fn(data);
        if (collectResults) results[myIdx] = result;
      } catch (e) {
        return reject(e);
      }
      runCount--;
      checkFinish();
      if (runCount < concurrency) readable.resume();
    });
    readable.on('close', () => {
      isOver = true;
      checkFinish();
    });
    readable.on('end', () => {
      isOver = true;
      checkFinish();
    });
    readable.on('error', reject);
  });
}
```