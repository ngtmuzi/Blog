---
title: promise的日常应用    
desc: promise在异步处理上真是比原始的回调好了太多，promise大法好~  
date: 2015-12-30 20:46:04.470
tags: 
- nodejs
- javascript
- promise
- express  
author: ngtmuzi  
category: 班门弄斧  
---
`promise`在异步处理上真是比原始的回调好了太多，`promise`大法好~

下面是一些日常工作中总结的各种神秘技巧：（使用`bluebird`模块）

* 一般来说都是在`request`请求或数据库操作之后开始使用`promise`，当然要直接`Promise.resolve()`也是可以的

* `mongodb`模块原生返回`promise`对象，真是方便不少，不过它用的是`Q`模块，没太多精力去了解模块间差异的我会选择在初始连接数据库的时候更换`promise`库：
```javascript
MongoClient.connect(mongoUrl, {promiseLibrary: Promise})
```
* `mysql`方面好像除了`sequelize`外没什么比较好的`promise`的模块……因为目前的工作内容都是在`mongodb`上，对这方面也没做太多了解

* `redis`模块中`ioredis`据说不错，不过也没太多接触

* 至于网络请求，自然是`request`的`promise`版：`request`-`promise`
---

当以上模块返回了promise对象，就可以用then一路走到黑啦

一般来说我的express路由处理函数都会以这样结尾：
```javascript
function getArticle(req, res, next) {
  mongo.article.find(req.query).sort({postTime:-1}).toArray()
    .then(res.ok, next);
}
```
`res.ok`是自己为了方便而挂上的一个函数，一般类似于  

```javascript
  res.ok  = res.json.bind(res);
```

`next`函数用于将错误传到路由处理函数的末端——错误处理函数，在那里进行统一的错误返回。

后来想想，用`next`函数统一处理错误固然看来很cooooooool，但也正因为此，对于各种错误的描述无法被带过去——用户并不在意你到底是数据库炸了还是代码写错了，他只想要一个中文的对错误的合理解释，类似“账号密码不正确！”之类的东西，因此考虑了一下，还是改成如下形式：

```javascript
res.ok = function (data) {
  res.send(200,data);
};
res.err = _.curry(function (code, err, ext) {
  res.status(code || 500);
  res.json({msg: err && err.message || err, ext: ext && ext.message || ext});
});
```
使用了`lodash`模块将函数柯里化——大概就是这样，虽然不太好说明原理，总之这是一种能让我这样处理回调的一种技术

```javascript
sms.csGetStatusReportExEx()
  .then(res.ok, res.err(502, '状态报告获取失败'));
```
真正的错误信息会被带在`ext`里字段返回，另外`res.ok`也改成调用适用性更好的`res.send`函数

（注意这里有个坑，`res.send`一个数字的话它会以为你只返回一个http状态码，因此虽然`res.send(200,data)`这种格式已被弃用，但是在目前还是必须这么处理才行）