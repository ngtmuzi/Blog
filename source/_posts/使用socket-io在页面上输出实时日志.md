---
title: 使用socket.io在页面上输出实时日志    
desc: 类似webshell的小玩具  
author: ngtmuzi  
category: 班门弄斧  
date: 2017-04-19 20:02:57  
tags: 
- node.js
- socket.io

---
### 需求

想在管理页面上看到日志文件/进程的实时输出日志，这对于某些不能直接ssh登录服务器，又没有良好的日志工具，或者单纯只是想检查运行输出的场景会比较有用

### 思路

#### 来

使用`tail -f`来得到实时的文件流或者用`pm2 logs`获取pm2进程的输出流，考虑为每个流加上一个`订阅id`，这样就可以实现多端同时订阅了

#### 去

在web页面上说到实时，自然就想到用`websocket`啦，逻辑并不复杂，客户端传来一个`订阅id`，就把它与id对应的流的`data`事件挂钩起来，这样便能将数据传递过去

### 核心代码

本文主要是给出一些实践的思路，所以不会有`socket.io`工作原理的说明

#### 产生流

使用`child_process.exec`运行子进程，该函数所返回的`stdout`属性就是一个可读流：

```javascript
const exec = require('child_process').exec;
//存储所有stream的集合
const streams = {};

/**
 * 用tail -f读文件
 * @param file
 * @return id
 */
function watchFile(file) {
  return watchProcess(`tail -f ${file}`);
}

/**
 * 自定义的命令行，比如pm2 logs 0
 * @param cmd
 * @return id
 */
function watchProcess(cmd) {
  return watchStream(exec(cmd).stdout);
}

/**
 * 对流做一些额外处理
 * @param stream
 * @return id
 */
function watchStream(stream) {
  const id     = Date.now();
  streams[id]  = stream;
  stream._buff = '';

  //处理流的data事件，使其按行(\n结尾)来触发自定义的line事件
  stream.on('data', data => {
    stream._buff += data;
    let lines    = stream._buff.split('\n');
    stream._buff = lines.pop();
    lines.forEach(line => stream.emit('line', line));
  });
}
```

#### 将流与socket.io订阅绑定
```javascript
  const io = SocketIO(httpServer);

  io.on('connection', function (socket) {
    socket.on('sub', function (id) {
      if (!streams[id]) return socket.emit('line', `该订阅id不存在: ${id}`);

      //管道函数，收到流的line事件则将控制台的ansi格式内容转成html格式然后触发客户端的line事件
      const pipe = line => socket.emit('line', ansiHTML(line));

      //订阅流的line事件
      streams[id].on('line', pipe);
    });
  });
```

#### 前端订阅id
```javascript
  var socket = io.connect();
  socket.on('line', function (data) {
    app.rawLines.push(data);
    if (app.rawLines.length > 2000) app.rawLines.splice(0, app.rawLines.length - 2000); //行数上限设为2000
  });

  socket.on('connect', function () {
    console.log('connect succeed');

    socket.emit('sub', 'time');//根据各种业务逻辑拿到一个订阅id并订阅
  });
```

前端页面为了方便渲染非常多行的元素，使用的是vue，代码就不贴了，最终效果如下

![](/img/webshell_1.png)

### 总结

这个东西功能接近于webshell了，最起码输出是可以显示了，再加个输入的接口就可以远程执行命令了，这里要提醒下允许远程执行命令是 **非常危险** 的。

主要的是思路，代码本身并不难，[demo已上传github](https://github.com/ngtmuzi/webshell-demo)，由于是demo，各种错误捕获和回收处理都不完善，这个还请自己研究了~

### 2017-06-14补充

监听逐行输出可以直接使用Node自带的`readline`模块，示例代码已更新