---
title: Proxy对象仿Scala的通配符函数调用  
desc: Proxy对象是ES6加入的新特性，使用它来监听拦截对象的操作，可以使我们完成很多原本javascript无法完成的特性  
date: 2016-4-23 16:47:40.471
tags: 
- javascript
- es6
- Proxy
author: ngtmuzi  
category: 神秘代码  
---
`Proxy`对象是ES6加入的新特性，使用它来监听拦截对象的操作，可以使我们完成很多原本`javascript`无法完成的特性，在不长的`scala语言`学习过程中，发现这门语言有一个很神奇的通配符`_`，在函数链式调用中，`_`就代表了对象本身，这使得函数式编程的语法更加简洁，放在javascript中举例，就类似这个样子：
```javascript
[1, 2, 3, 4].map(_.toString());
```
就是对每个传来的对象，调用他们自身的`toString()`方法，使用ES6中的`Proxy`对象，可以很容易地模拟该特性
（nodejs v6.0已支持`Proxy`，代码亦可在最新版chrome中运行）
```javascript
var _ = new Proxy({}, {
  get: function (target, key) {
    return function (obj) {
      if (obj && obj[key] && typeof obj[key] !== 'function') return obj[key];
      var args = arguments;
      return function (obj) {
        if (obj && obj[key] && typeof obj[key] === 'function')
          return obj[key].call(obj, ...args);
      };
    };
  }
});
```
除了调用对象自带的方法外，还有取出属性的功能，运行结果如下

```javascript
[1, 32, 128, 1024].map(_.toString('2'));
//return ["1", "100000", "10000000", "10000000000"]

[{v:123},{v:456}].map(_.v);
//return [123, 456]

Promise.resolve(new Date())
  .then(_.getTime())
  .then(_.toString())
  .then(_.length)
  .then(console.log, console.error);
//return 13
```
---
# 2016-6-8续
之前研究`Proxy`的时候写出来的那个`_`实际上还是有很大缺陷的，比如只能取一层的结构，参数的属性与方法同名时会取出参数的属性，不能正确捕捉到类型不对之类的错误，以及其他等等，于是就想做一个更完善的，现在想想当时的思路本身不太对，返回了一个函数就等于是限制了这个`_`的作用层数，不能继续往里面走，解决的方法当然是继续返回一个`Proxy`，思路如下：

* `_`本身返回一个Proxy，取属性操作返回的也是一个`Proxy`，这些`Proxy`指向一个函数，并在被调用时按顺序取出第一个参数里的属性

* `_`可以预接收参数，接收的参数将在`Proxy`被调用时传给取出来的函数

难点是如何知道`Proxy`是在预传参的时候被调用还是在真正取值的时候被调用，实际上用返回函数的方法可能很难解决这种问题，在再次深入研究了`Proxy`的相关资料后，我意识到上述的两种情况，对应的`this`应该是不同的，后来又发现过长的调用链每次都要经过多个`Proxy`可能会影响效率，我又改成另外一种形式实现：`Proxy`负责收集参数和属性名，并在最终调用的时候新建一个`Function`，将整个调用链还原回原本的格式（想到这里我就感觉坑了，实际上我只是把 `s=>s.xx.yy.zz()` 给简化成 `_.xx.yy.zz()` 而已，因此我特意加上了一个默认值的功能，好歹比箭头函数多了个错误捕获），再基于其他的一些需求，修修改改之后，变成了以下的格式：

```javascript
/**
 * Created by ngtmuzi on 2016/5/29.
 */
'use strict';
const receiveFn = (...args) => args;

/**
 * chain proxy maker
 * @params {Any} defaultValue
 * @type {Function}
 */
const chainProxy = module.exports = function (defaultValue) {
  var hasDefault   = arguments.length > 0;

  var handle = {
    get: function (target, property, receiver) {
      //set prototype to base Proxy Object _
      if (property === 'prototype') return _;
      if (property === 'apply') return target.apply;
      if (property === 'call') return target.call;
      if (property === 'bind') return target.bind;

      //is number
      if (!isNaN(+property)) return new Proxy(target.bind(null, `[${property}]`), handle);
      //return a new Proxy, go on
      else return new Proxy(target.bind(null, '.' + property), handle);
    },

    apply: function (target, thisArg, argumentsList) {
      //if method calling on Proxy Object
      if (thisArg && thisArg.prototype === _) {
        //save arguments to chain, and go on
        return new Proxy(target.bind(null, argumentsList), handle);

      } else {  //calling on outside

        //get the calling chain
        var chains = target();
        //pick arguments
        var args   = [].concat(...chains.filter(Array.isArray));

        //make function body
        var argNum     = 0;
        var expression = chains.reduce(function (a, b) {
          if (typeof b === 'string') return a + b;
          else if (Array.isArray(b)) return a + `(${b.map(()=> `args[${argNum++}]`)})`;
        }, 'return _');

        var fnStr = `
        try{
          ${expression};
        }catch(err){
          ${hasDefault ? 'return defaultValue;' : ''}
          err.stack = err.name + ': ' + err.message + '\\n\\tat: ' + '${expression}';
          throw err;
        }`;

        var finalFn = new Function(['args', '_', 'defaultValue'], fnStr);
        return finalFn(args, argumentsList[0], defaultValue);
      }
    }
  };

  var _ = new Proxy(receiveFn, handle);
  return _;
};
```
测试代码如下：
```javascript
var _1 = require('../index')(undefined);
var _2 = require('../index')();
var _3 = require('../index')({foo: 'bar'});

Promise.resolve({a: 12333})
  .then(_1.ab.toString().split('')[0].toString().replace('1', 'replaceStr').length)
  .then(console.log.bind(null, 'result 1:'), console.error.bind(null, 'catch error1:'))

  .then(()=>({a: 12333}))
  .then(_2.ab.toString().split('')[0].toString().replace('1', 'replaceStr').length)
  .then(console.log.bind(null, 'result 2:'), console.error.bind(null, 'catch error2:'))

  .then(()=>({a: 12333}))
  .then(_3.ab.toString().split('')[0].toString().replace('1', 'replaceStr').length)
  .then(console.log.bind(null, 'result 2:'), console.error.bind(null, 'catch error2:'));
```
本段代码已经做为模块提交到[npm](https://www.npmjs.com/package/chainproxy)上