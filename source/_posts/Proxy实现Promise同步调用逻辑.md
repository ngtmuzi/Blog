---
title: Proxy实现Promise同步调用逻辑  
desc: Proxy + Promise = 逻辑同步  
date: 2016-8-8 23:30:08.830
tags: 
- javascript
- Proxy
- Promise
- ES6
author: ngtmuzi  
category: 神秘代码  
---
　　我们在很多使用Promise的时候都会有如下的用法
```javascript
MongoClient.connect()
  .then(function (db) {
    var col = db.collection('something');
    col.find().toArray()
      .then(console.log, console.error);
  });
```
　　真正的连接对象在`connect()`之后才返回，只能在`then`回调中处理后续的数据库操作，而如果想复用这个连接，就需要在外面用另一个对象保持对它的引用，代码可能就会变成这个样子
```javascript
var db;
MongoClient.connect()
  .then(function (_db) {
    db = _db;
  });
//别的地方
db.collection('otherthing')....
```
　　这样的缺点是在连接未建立时访问`db`会引发异常，当然最正确的做法应该是
```javascript
var db = MongoClient.connect();
//别的地方
db.then(function (_db) {
  _db.collection('otherthing')....
});
```
　　可以保持对一个连接的复用，但这样还是嵌了一层回调。

　　在之前用`Proxy`写了几个玩具之后，有个念头渐渐浮上心头，`Proxy`也许可以改变这个模式，我们可以用`Proxy`预先收集好调用链，然后再将其内部转为`Promise`的调用链，简单来说思路如下：

1. 先将原始数据用`Promise.resolve`包裹，使其成为一个`Promise`对象，并返回该对象的代理`Proxy`

2. 对该`Proxy`的`get`和`apply`操作实际上就是在后面多加一个`.then`回调，并会继续返回`Proxy`，以实现链式调用

3. 当将`Proxy`作为函数运行时对参数预先做`Promise.resolve`处理，使得它可以接受`Promise`参数

4. 在内部的取值、调用时做错误捕获，使得在冗长的调用链中捕获错误成为可能

5. 最终的值从`.then中`获取，`Promise`的特性保证了最终值的可达性


　　多说无用，以下是一些示例：

```javascript
var request = require('request-promise');
var _       = require('promixy');

var parse   = _(JSON.parse);            //proxy a function
var docJson = request('https://nodejs.org/api/documentation.json');  //a promise 
var obj     = parse(docJson);           //wait the promise result

obj.miscs[0].textRaw.then(console.log, console.error);
```
　　首先代理了`JSON.parse`使其可以接受`Promise`类型的参数，并挂上长长的一串取属性操作，得到的结果最后通过`console.log`打印出来

　　当试图从`undefined`中读取属性时，会返回类似这样的错误
```
TypeError: can't read property 'textRaw' in {Promixy}(1 args).miscs.10, it got a undefined
```
　　虽然有些不直观但好歹调用链是能看出来了
```javascript
_(Promise.resolve(_(_(_(111))))).then(_(console.log), _(console.error));
//111
```
　　即使嵌了很多层，这个`Proxy`依然是`Promise`的代理，所以最后还是可以得到正确的结果的

　　而刚才的mongo连接也可以简化为

```javascript
var db = _(MongoClient.connect());
//别的地方
db.collection('something')...
```
　　实质上是`Proxy`在内部代替用户嵌了一层`.then`，代码上更加简单直观，且从它上面取的所有属性依然有`Proxy`包裹，所以你甚至可以实现这样变态的调用方式

```javascript
db.collection('something').find().toArray()[0].somekey
  .then(console.log,console.error);
```
　　直接取出集合第一条中的某个值，当然我不建议这种很难理解的写法

　　综上，这个模块还有许多待发现的可用之处，我个人也将它用于了生产环境中，[npm地址](https://www.npmjs.com/package/promixy)，如果你有任何建议或发现了bug，欢迎到[github](https://github.com/ngtmuzi/promixy)反馈