---
title: NodeJS:Stream研究笔记    
desc: 相信学过nodejs的人都必然会接触到nodejs中的流（stream） 
date: 2016-2-15 21:00:21.199
tags: 
- nodejs
- stream  
author: ngtmuzi  
category: 神秘代码
---
　　相信学过`nodejs`的人都必然会接触到`nodejs`中的流（`stream`），不提从`fs`、`net`、`http`这些基础模块，到`express`、`request`、`mongodb`这些常用模块，处处都有流的身影。在初学时我也曾被`pipe`方法的强大特性吓到，参看`request`的文档：

```javascript
request('http://google.com/doodle.png').pipe(fs.createWriteStream('doodle.png'));

fs.createReadStream('file.json').pipe(request.put('http://mysite.com/obj.json'));

request.get('http://google.com/img.png').pipe(request.put('http://mysite.com/img.png'));
```
　　从网络流到文件流的转换自然直观，甚至可以以此实现最简单的代理服务器或静态文件服务器。最近在研究一些文件上传的东西，就重温了一下流，才发现其中大有乾坤。

　　流在`nodejs`内部提供的方法都是事件式的，需要用`on`方法将我们的回调函数挂在相应的事件上，如`close`、`end`、`data`、`drain`等，这种形式与`nodejs`异步的惯用套路——callback相差甚远，事件响应式编程也是一种可行的异步编程解决方案，不过势必会造成逻辑被不同的事件响应函数所分割，有时事件触发的先后顺序也会引起困扰，当然这不是我要讨论的重点。

　　流的`pipe`方法在初学者的我看来简直是万能钥匙，只要搞到一个可读流和一个可写流，一个`pipe`就完事了，里面具体是什么原理都不用管。不过在看过源码后才发现`pipe`也不是很复杂，主要的业务代码如下

```javascript
Stream.prototype.pipe = function(dest, options) {
  var source = this;

  function ondata(chunk) {
    if (dest.writable) {
      if (false === dest.write(chunk) && source.pause) {
        source.pause();
      }
    }
  }

  source.on('data', ondata);

  function ondrain() {
    if (source.readable && source.resume) {
      source.resume();
    }
  }

  dest.on('drain', ondrain);

};
```
　　就是简单的对流的事件的应用，当可写流（ws）处于可写时，打开可读流（rs），当rs的数据读进来了，就往ws写，写到满了就暂停读入rs，就是这样一种循环，至于具体不同种类的流之间的细节，全都被封起来了。

---
　　以上是最基本的流的原理，而在不同的模块中，流也有很多有趣的用法：

* `net.createServer`方法接收一个函数，用于获得`socket`套接字对象，而它就是一个可读可写流对象，我们需要自行处理它所接收的数据，因为TCP流之间的数据没有间隔，所以如何从连续的流中获取一个完整的消息包，也是很多人纠结的问题，常用的方法是定义包的前几位作为包头，在其中定义整个包的长度，或者干脆指定固定的包长度之类的，需要注意的是每次`data`事件收到的数据需要好好保存，它其中可能包含多个包，也有可能只是一个包的一小部分，想要在底层搞个应用也不容易啊……

* `http`也是跟流打交道很多的模块，比如`http.request`函数本身就返回一个可写流，而它的回调函数又接受一个可读流，而且需要在可写流（也就是http请求流）写完之后，才会真的去进行http请求，获取到http头之后，再传给回调函数一个可读流，之后的http正文才从可读流中流过来，如果中途出错，会通过`error`事件抛出。可见此过程很不直观，但在众多的err开头的回调函数中，也很新奇。

```javascript
var req = http.request('http://baidu.com', function (res) {
  console.log(res.headers);     //调用回调的时候，已经获取到了http头
  res.on('data', console.log);  //读http正文（表单部分存储于此）
});
req.end('hello');  //只有请求流写完之后，才后真正进行http请求
````
* fs模块的文件流有一个很实用的特性，那就是指定读写位置，如`fs.createReadStream`函数可以指定文件读取的开始和结束位置，由此可以联想到最直接的用法，就是与http服务配合，作为静态文件服务器，以提供文件的多线程下载、断点续传等功能，`express`模块中负责静态文件服务的`send`模块，大多数代码负责从http请求头中获取文件的范围（Range）和查找文件，而最核心的代码只有这一句
```javascript
// pipe
var stream = fs.createReadStream(path, options)
stream.pipe(res);
```
* 而写文件流也可以指定写入的起始位置，可以很自然地想到，这对于前面的断点续传等功能有多大的帮助。这里值得一提的是`options`中的`flags`参数，当取默认值'w'时，它每次都会重写文件，而改成'r+'，就不会将文件清空。

* `mongodb`的node模块为配合`GridFS`功能，也使用了流的特性来进行文件读写，参见`GridFSBucket`类。

　　在使用各式各样流的时候，要记得的一点是，错误不再位于回调参数的第一位，需要用`on('error')`来自行捕获，之后就可以愉快地使用各种`pipe`了~