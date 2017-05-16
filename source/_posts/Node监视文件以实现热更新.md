---
title: Node监视文件以实现热更新
desc: 在有限范围内使用效果还是很好的
author: ngtmuzi
category: 神秘代码
date: 2017-05-16 11:47:16
tags:
- nodejs
---

在准备开始写的时候搜了一下相关的文章，看到了这篇fangshi的[《Node.js Web应用代码热更新的另类思路》](http://fex.baidu.com/blog/2015/05/nodejs-hot-swapping/)，写得很详细考虑得也很全，我的思路也类似这样，不过在替换旧模块上有些不同，总结出来权当抛砖引玉。

为了方便说明，部分代码有省略细节，详细可以参见[完整代码](https://github.com/ngtmuzi/wheel/blob/master/tools/watchModule/index.js)

## 需求

我的最开始需求倒不是要实现热更新这样听起来很炫酷的功能，只是想动态更新配置文件（JSON或JS）的内容到内存，避免每次改小小的配置都要重启进程

## 基本思路
* 监视文件/目录改动
* 清空require.cache中的模块缓存并重新require
* 用新模块覆盖旧模块

## 监视文件/目录改动

### 定位
首先使用`path.resolve`来定位文件，其实使用`require.resolve`可以根据[node寻找模块的规则](https://nodejs.org/api/modules.html#modules_file_modules)更智能地定位到一个模块的入口文件（比如xxx/index.js）的，但更多情况下我们并不只是监视这个index.js而是想监视整个文件夹的改动（举个例子，index.js里require了同目录的xx.json并做了一系列计算最后把计算结果挂载`module.exports`上，这个时候单单监视index.js是没什么用的。）

### 监视/防抖动
原本是简单地使用`fs.watch`来监视文件，但其在linux下是无法监视到子目录/文件的改动的（参见[node文档](https://nodejs.org/api/fs.html#fs_caveats)），因此后来改用了被众多知名工具依赖的文件监视模块[chokidar](https://github.com/paulmillr/chokidar)，并且出于实际情况增加了防抖动
```javascript
chokidar.watch(filePath).on('all', lodash.debounce(update, 300));
```

## 清空require.cache中的模块缓存并重新require

### 清空缓存
考虑到监视的有可能是一个目录而非单个文件的情况，我们需要在清除时多考虑一下，把整个目录的引用都清除掉
```javascript
Object.keys(require.cache).forEach(function (cachePath) {
  if (cachePath.startsWith(filePath)) {
    delete require.cache[cachePath];
  }
});
```

### 重新require
```javascript
var newModule = require(filePath);
```
这个时候可能会报一些找不到文件，代码语法错误之类的同步错误，这个属于预期范围内，我的处理逻辑如下：
* 第一次require是同步的，这时的错误会同步抛出，一般来说就会结束进程，因为确实没找到文件
* 监视事件触发并重新require时产生的错误会丢给回调函数，并且保持原模块的内容不做更改（避免意外修改文件产生语法错误导致模块失效或进程退出）

## 用新模块覆盖旧模块
如果我们在使用模块时能够遵守一个约定：**`module.exports`是Object，且其他模块永远从该模块所暴露的`module.exports`上取值**，那么我们就不需要去做反射，闭包之类的处理，只要简单地使用
```javascript
Object.assign(target,newMoudle)
```
就可以在保持该对象的引用不变的情况下增改属性，考虑到有删除属性的情况，我自己写了一段比较暴力的覆写的函数
```javascript
function override(target, source) {
  Object.keys(target).forEach(function (key) {
    if (!source.hasOwnProperty(key)) delete target[key];
  });
  Object.keys(source).forEach(function (key) {
    target[key] = source[key];
  });
  return target;
}
```
外部模块只要是遵守了上述约定，就可以完全透明地取得最新的属性内容，对于我主要的应用场景——动态读取配置文件来说，这个还是很容易遵守的

## 使用方式
只有一个模块引用的话，直接调用即可
```javascript
const some = watchModule('./originModule');
//从module.exports上取得的一定是最新值
console.log(some.a);
```
当有多处需要引用时，建议使用一个代理的模块来挂载，这样在其他模块就可以直接用普通的require了（注意不要对一个模块多次调用watchModule，这样会产生重复事件）
```javascript
//originModule
module.exports={a:1}; 

//代理模块
module.exports = watchModule('./originModule');

//其他模块
const some = require('proxyModule');

//从module.exports上取得的一定是最新值
console.log(some.a);
```
具体到上面文章提到的express动态挂载路由，`app.use`需要的是一个函数，因此我们无能为力——原函数已经被`app.use`挂载到中间件链上了，这种情况还是考虑使用一层闭包吧

## 总结
思路都是类似的，只是我多加了一个约定，只要遵守这个约定我们就可以写出一个比较通用的监视模块，当然这也并不是万能的，比如module.exports必须是Object（其他类型可以用Object多包裹一层），很多极限条件也没考虑到（比如Proxy、不可变Object、原型链、不可枚举的属性等），但对于普通的业务代码和配置文件来说这应该是没有什么问题了  

另外提醒一点，允许动态更新代码是**非常危险**的，比如我提到的这种允许读js作为配置文件的情况，万一js里来句`process.exit`或者其他恶意代码就挂了，可以根据实际需要来考虑加上限制

[完整代码](https://github.com/ngtmuzi/wheel/blob/master/tools/watchModule/index.js)，欢迎讨论指正。

## 补充：旧模块资源的释放
阅读了上面的文章后才发现确实没考虑到这里，并且由于配置文件并不是频繁改动，在正式环境下也没出现过问题，但测试过后确实存在旧模块没有释放的情况（考虑还是不周啊），我们可以参考上面文章中fangshi给出的代码来清除引用
```javascript
    var module = require.cache[modulePath];
    // remove reference in module.parent
    if (module.parent) {
        module.parent.children.splice(module.parent.children.indexOf(module), 1);
    }
```
