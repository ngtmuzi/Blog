---
title: javaScript的函数柯里化  
desc: 柯里化（Currying）是把接受多个参数的函数变换成接受一个单一参数(最初函数的第一个参数)的函数  
date: 2016-1-7 10:11:18  
tags: javascript  
author: ngtmuzi  
category: 班门弄斧  
---
> 柯里化（Currying）是把接受多个参数的函数变换成接受一个单一参数(最初函数的第一个参数)的函数，并且返回接受余下的参数且返回结果的新函数的技术。这个技术由 Christopher Strachey 以逻辑学家 Haskell Curry 命名的，尽管它是 Moses Schnfinkel 和 Gottlob Frege 发明的。

　　简单理解来说就是把一个函数带上经常用到的参数和上下文，生成一个新的函数的技术，lodash的柯里化函数写得好复杂看得不是很懂，自己想了一下应该不是很复杂才对：
```javascript
function curry(fn, context) {
    function c() {
        var args = this.concat(Array.prototype.slice.call(arguments));
        if (args.length >= fn.length) return fn.apply(context, args);
        return c.bind(args);
    }

    return c.bind([]);
}
```
　　返回一个用于收集参数数组的函数，当参数达到原函数的参数长度时才调用原函数，参数数组传到新函数的this对象上以供调用，测试如下：

```javascript
var _a = function (a, b) {
    return a + b;
};
var a = curry(_a);

console.log(a(1, 2), a(2)(3), a()()(3)(4), a()()(4, 5))
//运行结果： 3 5 7 9
```