---
title: Node处理字符编码相关经验
desc: 在UTF8与GBK之间摸爬滚打一段时间后摸索出来了一些门道  
date: 2016-5-25 10:10:55.618
tags: 
- nodejs
- encoding
author: ngtmuzi  
category: 神秘代码  
---
在UTF8与GBK之间摸爬滚打一段时间后摸索出来了一些门道：

首先要确定一点原则，就是尽可能往UTF8转，因为Node本身是用UTF8的，需要转成其他编码的时候再用`iconv-lite`就行了


自动判断文本编码并解码为UTF8
---
```javascript
var iconv     = require('iconv-lite');
var jschardet = require('jschardet');
function readText(filePath) {
  var fileData = fs.readFileSync(filePath);
  var encoding = jschardet.detect(fileData).encoding;
  if (!iconv.encodingExists(encoding)) encoding = 'utf8';
  return iconv.decode(fileData, encoding);
}
```
`jschardet`模块的原理不用看源码都能大概猜到，是判断文本二进制码的范围，因为不同的编码占用的区块是有差异的，得到编码之后再用`iconv-lite`转成UTF8即可


读取GBK编码页面
---
```javascript
var request = require('request-promise');
var iconv   = require('iconv-lite');
request('http://someurl.com', {encoding: null})
  .then(s=>iconv.decode(s, 'gbk'))
  .then(console.log, console.error);
```
将`encoding`指定为`null`的时候`request`模块会返回一个`buffer`，我们将其手动gbk解码即可


处理gbk编码的urlencode
---
```javascript
var iconv   = require('iconv-lite');
var gbkStr = '%c4%e3%ba%c3';
iconv.decode(new Buffer(gbkStr.replace(/%/g, ''), 'hex'), 'gbk');
```
`express`在碰到无法解码的urlencode的时候会将其保留为原格式，好在urlencode就是16进制前面加了%号，因此我们可以手动转成`buffer`之后再解码



转发无法被解码的urlencode
---
假设我们的程序只是做纯粹的转发，没想到参数里有不能解码的urlencode，比如上面的`%c4%e3%ba%c3`，这个时候再通过`request`去请求的时候，再次做了urlencode，会把原字符串里的百分号转成`%25`，那么目标方接到参数的时候就变成`%25c4%25e3%25ba%25c3`，为了避免这一点，我们需要对无法被urlencode解码的部分做无伤转发，示例如下
```javascript
var qs   = require('querystring');
var data = {gbkStr: '%c4%e3%ba%c3'};

function urlEncode(s) {
  return qs.escape(s.replace(/%(\w{2})/g, '-_-$1')).replace(/-_-(\W{2})/g, '%$1');
}

//1、使用原生的queryString
var url = 'someurl' + qs.stringify(data, null, null, {encodeURIComponent: urlEncode});

//2、直接用于request
request('someurl', {
  qs                : data,
  qsStringifyOptions: {options: {encodeURIComponent: urlEncode}}
}).then(console.log, console.error);
```
关于`querystring`更多的细节参看[官方文档](https://nodejs.org/api/querystring.html#querystring_querystring_stringify_obj_sep_eq_options)

我们可以让`querystring`结合第一段代码的自动判断编码，来实现一个自动化的解码流程，当然越是自动化的东西，就越难搞清楚内在的原理，在很多时候，还是自己来实现比较靠谱