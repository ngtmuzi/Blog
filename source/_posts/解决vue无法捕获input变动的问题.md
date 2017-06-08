---
title: 解决vue无法捕获input变动的问题
desc: 手动触发Input事件就好了
author: ngtmuzi
category: 神秘代码
date: 2017-06-05 13:35:42
tags:
- vue
- jquery
- javascript
---
## 问题

自己的页面使用了vue和一款jquery的时间选择插件`datepicker`，但选好时间后input框的变动并没有被vue捕获到

## 参考资料

> [Vue表单控件绑定](http://cn.vuejs.org/v2/guide/forms.html#基础用法)  
> [Vue源码](https://github.com/vuejs/vue/blob/dev/dist/vue.js#L6090)  
> [Jquery的val()方法说明](http://api.jquery.com/val/#val-value)  
> [从vue.js的源码分析...](http://www.cnblogs.com/Eden-cola/p/vue-v-model-with-input.html)  
> [MDN: Event对象](https://developer.mozilla.org/zh-CN/docs/Web/API/Event)

## 分析

jquery的val方法文档写得很清楚：

> Setting values using this method (or using the native value property) does not cause the dispatch of the change event.

在类似使用`jquery`的`val()`方法时或者原生dom方法改动input的值时，不会触发任何事件。

> [vue.js #L6090](https://github.com/vuejs/vue/blob/dev/dist/vue.js#L6090)
 ```javascript
  var event = lazy
    ? 'change'
    : type === 'range'
      ? RANGE_TOKEN
      : 'input';
```

而vue是通过绑定事件（默认是`input`）来获知表单元素的变动的，因此这个问题就是因为插件直接用`.val()`设置了值，而vue无法获知。

## 解决方案

我们可以在改动值后手动触发`input`事件，注意这里我们需要使用DOM原生的`event`对象，而不能直接使用jquery的`.trigger()`方法
```javascript
//.get(0)返回DOM原生对象
$('#input1').val('something').get(0).dispatchEvent(new Event('input'));
```
在我这个具体问题中，监听插件的自定义事件就可以了
```javascript
$(".form_datetime").datepicker()
  .on('changeDate', function (e) {
    //触发DOM对象上的原生input事件
    this.dispatchEvent(new Event('input'))
  });
```
注意如果input使用的修饰符带有`.lazy`则应该触发`change`事件

当然最好的办法还是避免使用其他库来改动表单，改为使用基于vue的插件之类的，省心省力