---
title: 个人使用Elastic+Kibana的一些经验
desc: 在记录日志的时候引入了elk栈，但日志写入部分是自己写的，并没有使用`logstash`，因此经验就不包括`logstash`啦
author: ngtmuzi
category: 班门弄斧
date: 2017-07-17 19:03:51
tags: 
- ElasticSearch
- Tool
---
在记录日志的时候引入了elk栈，但日志写入部分是自己写的，并没有使用`logstash`，因此经验就不包括`logstash`啦

## ElasticSearch部分

### 有一个管理后台

ElasticSearch（以下简称ES）本身不带GUI，官方插件集x-pack添加到kibana中的monitoring界面只能看索引/集群信息却不能做管理，在你想删除/关闭/合并分段等操作的时候就会感觉束手无策，用RESTful的工具（如Chrome的插件PostMan）来请求ES的接口固然是一种方法，但毕竟不是长久之策，除非你喜欢这么玩……

目前Github上最受欢迎的项目是[elasticsearch-head](https://github.com/mobz/elasticsearch-head)，功能完善足以满足需求，不过在一些小细节不是很尽如人意……比如它是纯前端去访问ES接口的，会碰到跨域问题，官方提供的解决方法如下：
* 去改ElasticSearch的配置允许跨域
* 开一个允许跨域的本地代理（项目自身有提供）
* 安装一个插件到ES上

根据自己项目的实际情况也可以自己做一个后台实现一些简单的功能，类似这样

![](/img/elk_desc_1.png)

### 注意资源使用量

数据量大而资源不足的情况下要尤其注意这点，即使按照ES官方文档的推荐设置好配置，也给了32G大内存，ES还是会在某些深夜默默地GC超时然后内存溢出崩溃，以下是一些建议：
* 最好使用3台及以上的机器组成集群来使用
* 负责接受写入请求的节点以及主节点最好不存数据，避免这个节点崩溃后上游调用方全都写不进数据
* 定时清理内部的索引，一般推荐的索引维度是按天分索引，因此可以做一些通配条件来对一定时间前的索引做删除/关闭/合并分段操作减少内存占用

### 注意文档字段的类型

向ES写入数据时，要注意JSON内各字段的类型，ES会“智能”地以字段第一次出现时的类型来动态建立Mapping（参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/5.2/dynamic-field-mapping.html)），并且不能在之后改变。假设`a`字段第一次传的是字符串，而第二次传了数值，ES会返回一个Mapping不匹配的错误，另外如果你传的字符串第一次刚好符合日期的格式，那么这个字段就被认为是Date类型，下次传其他字符串的时候也会返回Mapping错误，这点要注意避免

## kibana部分

### lucene语法

[官方文档](http://lucene.apache.org/core/6_6_0/queryparser/org/apache/lucene/queryparser/classic/package-summary.html)，个人的使用经验是遇事不决用括号`()`，很多时候搜索结果不符合预期是因为没有括号导致解析器误解了搜索条件

### 慎用通配符

在搜索字符串两侧都加`*`可能会有严重的性能问题，这点要小心

### 导出数据到Excel

虽然kibana本身的图表功能已经非常强大了，但总是有些需求要你导出数据到excel等工具上做进一步的分析，注意在图表的左下角有箭头，展开之后就能看到导出数据的按钮了

![](/img/elk_desc_2.png)