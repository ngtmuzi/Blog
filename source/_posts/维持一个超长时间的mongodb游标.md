---
title: 维持一个超长时间的mongodb游标
desc: 又在发废文
author: ngtmuzi
category: 班门弄斧
date: 2020-06-17 18:29:46
tags: 
- mongodb
---

**本文使用的是node.js版本的mongodb连接库**

写这篇的时候才发现我根本没写过`tag`是 [#mongodb](https://ngtmuzi.com/tags/mongodb/) 的博客，足以见得知识浅薄，甚至这篇也没有什么技术含量

有个有趣的工作是遍历某mongo库的索引，对其上符合要求的文档做某些操作，数据量大到可能要跑十天半个月，中途可能还要避开业务高峰期，而每次都用`find({_id:{gt:lastid}}).sort().skip().limit()`感觉又不够好玩

于是计划用一个游标从头遍历到尾，这样的好处是不会重复发出多次op，在网络请求层面上还是要好一点的（游标有一个`batchSize`控制每次读取的数据量，可以视情况调整），而我又可以使用游标的流特性，来做一些下游打包、管道之类的好玩操作（见[《Stream研究笔记II》](https://ngtmuzi.com/NodeJS%EF%BC%9AStream%E7%A0%94%E7%A9%B6%E7%AC%94%E8%AE%B0II/)）

## 延长查询超时时间

> [官方文档-MongoClient](http://mongodb.github.io/node-mongodb-native/3.5/api/MongoClient.html)

`MongoClient`建立实例时传递的`socketTimeoutMS`选项默认是6分钟，当查询耗时非常长时（比如没命中索引的`count()`），客户端将不再等待服务端返回并抛错，此选项只是以防万一，如果没有耗时很长的单次查询，这个可以保持默认

## 设置游标不超时

> [官方文档-Collection.find()](http://mongodb.github.io/node-mongodb-native/3.5/api/Collection.html#find)

在`find()`的参数内传递`noCursorTimeout`，避免闲置的游标被服务端主动释放

## 设置session活跃

然而只靠上面的选项还不够，[官方文档-cursor.noCursorTimeout](https://docs.mongodb.com/manual/reference/method/cursor.noCursorTimeout//) 又说了，服务端还是会清理闲置30分钟以上的`session`（每个`op`都包含在`session`中，不主动指定的话会生成一个隐式的），除非你主动地定期刷新`session`的活跃状态，连代码都附出来了

```javascript
var session = db.getMongo().startSession()
var sessionId = session.getSessionId().id

var cursor = session.getDatabase("examples").getCollection("data").find().noCursorTimeout()
var refreshTimestamp = new Date() // take note of time at operation start

while (cursor.hasNext()) {

  // Check if more than 5 minutes have passed since the last refresh
  if ( (new Date()-refreshTimestamp)/1000 > 300 ) {
    print("refreshing session")
    db.adminCommand({"refreshSessions" : [sessionId]})
    refreshTimestamp = new Date()
  }

  // process cursor normally

}
```
需要自己显示声明一个`session`，在它之上发起游标查询，然后定期刷新`session`，这样游标就不会被服务器给释放掉了
