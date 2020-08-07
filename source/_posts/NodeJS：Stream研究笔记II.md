---
title: NodeJS：Stream研究笔记II
desc: 主要是Node源码阅读理解
tags: 
- nodejs
- stream  
author: ngtmuzi
category: 班门弄斧
date: 2020-04-05 20:19:39
---
几年前写的[研究笔记](/NodeJS：Stream研究笔记)现在回头看感觉全是在抄官方文档里的东西，近期真正在写stream的时候又发现其实根本没算了解，都没自己实现过读流和写流，于是补充一篇。

以下内容主要来自官方文档及自行阅读源码，较多内部逻辑及细节已被省略，只抽象其主干思想：

## Readable可读流

> [官方文档-实现一个可读流](https://nodejs.org/dist/latest-v12.x/docs/api/stream.html#stream_implementing_a_readable_stream)

* 所有可读流都必须提供`_read()`方法的实现，以从基础资源中获取数据。
* 当调用`_read()`时，如果资源中已有可用数据，则应该使用`this.push(dataChunk)`将数据加入到读流的缓冲区队列中，然后`_read()`应该持续以上操作直到`push()`返回`false`。

![](/img/streamII-1.png)

从结论来说，如果你不调用`push()`，可读流永远不会产生数据。

关于可读流的两种读取模式在这里就不再赘述了，有趣的是可读流内部的缓冲区`buffer`并不是一个`Buffer`对象（因为流要支持对象模式，不能只支持二进制数据）或者`Array`，而是一个自己实现的单链表，带头尾指针，这里应该是考虑方便缓冲区的前后读写，并且不需要使用数组的索引功能。

`push()`函数还提到说当它返回`false`时就不应该继续写入了，其实代码内部中只是返回了缓冲区与`highWaterMark`的大小对比，其实并不会抛弃超过的数据，所以自己实现时多写一些内容进去是没问题的（比如一次批量拿到了N个数据，总不能写一半丢一半）。

## Writable可写流

> [官方文档-实现一个可写流](https://nodejs.org/dist/latest-v12.x/docs/api/stream.html#stream_implementing_a_writable_stream)

* 所有可写流实现必须提供`_write`方法，以将数据发送到基础资源
* 必须调用回调方法来表示本次写入的成功或失败
* 在调用`_write()`和回调被调用之间如果有新的`write()`调用，则这些数据会先被放入缓冲区。调用回调时，流可能会发出“`drain`事件。

![](/img/streamII-2.png)

可写流相对比可读流逻辑简单一点，就是包裹一下用户自定义的写方法，并且做缓冲队列，从这里我们能知道很重要的一点：可写流的`write()`是串行调用的，不支持并发，因此才有了一个`writev()`方法作为批量写入的替代。`write()`返回的`true/false`也仅是表示数据量是否达到了它的水位线，但继续向其写入也是没问题的（当然要注意内存用量）。

## pipe方法

结合上面两张图之后再去看`pipe()`方法就会比较好理解了

![](/img/streamII-3.png)

## Duplex双工流

> [官方文档-实现一个双工流](https://nodejs.org/dist/latest-v12.x/docs/api/stream.html#stream_implementing_a_duplex_stream)

就是简单继承了可读流和可写流而已，注意它并不是像一根水管一样一边写了一边就可以读，需要用户自己去实现读写方法，换言之你可以让流入和流出的数据完全无关。双工流与分开实现一个可读流一个可写流的区别，主要在于它可以帮你管理一些流的状态，比如`allowHalfOpen`不允许半开，这样读流触发`end`事件（如`push(null)`）时会去关闭写流。不过我们一般需求的场景是反过来的：写流end后就结束读流，但双工流无法实现，因为它没有与你约定好中间的数据流转逻辑，不知道读流何时才能正确被结束，因此我们一般使用它的子类`Transform`转换流。

## Transform转换流

> [官方文档-实现一个转换流](https://nodejs.org/dist/latest-v12.x/docs/api/stream.html#stream_implementing_a_transform_stream)

* `_transform()`可以当作是可写流的`_write()`的用途
* `_flush()`是上游流发出`end`事件时触发的方法，可以在这里将残余的中间数据处理并输出到下游去

转换流是双工流的子类，封装了一些转换的逻辑，主要思路就是：

可写流`write()`接收数据 => _transform()方法 => push()方法 => 可读流产生数据输出

因为它本质上还是对可读流、可写流的封装，因此类似可写流的`highWaterMark`与串行写入实现的流速控制、`pipe()`管道方法等特性都是可以享受的，Node内常见的对流的加密压缩也都是用它实现的，注意的一点是`_transform()`方法本身是不会帮你把数据`push()`到可读流的（只有用户自己知道什么时候数据才是可以被输出的），因此要注意自己保留中间数据、及时调用`push()`输出、以及在`_flush()`接口内处理残余数据。

![](/img/streamII-3.png)

这里附一个简单易懂的例子：

```javascript
const stream = require('stream');

/**
 * 将逐条数据缓存在内存，达到batchSize或上游流结束时，才向下游发出一条打包的data消息
 */
class BatchStream extends stream.Transform {
  /**
   * @param {*} batchSize 批量大小
   * @param {*} options 传递给基类stream的options，部分属性固定
   */
  constructor(batchSize = 500, options = {}) {
    options.allowHalfOpen = false;
    options.writableObjectMode = true;
    options.readableObjectMode = true;
    options.highWaterMark= 1024;
    super(options);
    this.batchSize = batchSize;
    this._chunks = [];
  }
  async _transform(chunk, encoding, callback) {
    this._chunks = this._chunks.concat(chunk);

    while (this._chunks.length >= this.batchSize) {
      const batch = this._chunks.splice(0, this.batchSize);
      this.push(batch);
    }

    callback();
  }
  _flush(cb) {
    if (this._chunks.length) this.push(this._chunks)
    cb();
  }
}
module.exports = BatchStream;
```

刚开始编写流的实现时可能要注意的一点是，流本身带有的那些数据缓冲区、写入队列并不是给你用的，因此如果要你保留一些中间状态，必须要自己建变量存着，比如上面例子的`this._chunks`。

## 总结

Node的流设计与Promise一样，本质上都是对回调、异步控制的封装和抽象，由于基于事件订阅的机制实现，它可以实现流速控制，更适合用于持续的异步过程，并可用`pipe()`管道方法将其简化，在使用中有以下点可以关注：

* 可读流`on('data',cb)`形式订阅数据事件的形式，无法直接控制并发量，可以考虑用类似[streamQueue](/Node对流的Promise包装和并发控制/)这样的形式再做一层控制，或者自己实现一个可写流，使用`pipe()`来做自动的流速控制
* 可写流的`write()`实际是串行处理写入数据，本身虽然保证了数据的有序性，但不一定高效，在追求效率的时候，可以考虑实现`_writev()`接口，或者不在`_write()`内等待异步执行，直接回调`cb`，如果出现问题想要中断流程，可以再手动触发错误`ws.emit('error')`

