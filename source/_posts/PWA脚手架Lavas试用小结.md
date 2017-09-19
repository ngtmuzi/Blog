---
title: PWA脚手架Lavas试用小结
desc: 来自百度的vue+pwa解决方案
author: ngtmuzi
category: 随笔
date: 2017-09-18 16:48:14
tags:
- 前端
- vue
- Tool
---

## 简介

基本摘自[Lavas官网](https://lavas.baidu.com)

### [什么是PWA](https://lavas.baidu.com/doc)

Progressive Web App, 简称 PWA，是提升 Web App 的体验的一种新方法，能给用户原生应用的体验。

PWA 能做到原生应用的体验不是靠特指某一项技术，而是经过应用一些新技术进行改进，在安全、性能和体验三个方面都有很大提升，PWA 本质上是 Web App，借助一些新技术也具备了 Native App 的一些特性，兼具 Web App 和 Native App 的优点。

PWA 的主要特点包括下面三点：

* 可靠 - 即使在不稳定的网络环境下，也能瞬间加载并展现
* 体验 - 快速响应，并且有平滑的动画响应用户的操作
* 粘性 - 像设备上的原生应用，具有沉浸式的用户体验，用户可以添加到桌面

### [Lavas 是什么](https://lavas.baidu.com/guide)

Lavas 是一个基于 Vue 的 PWA (Progressive Web Apps) 完整解决方案。我们将 PWA 的工程实践总结成多种 Lavas 应用框架模板，帮助开发者轻松搭建 PWA 站点，且无需过多的关注 PWA 开发本身。

## 安装
```cmd
npm install -g lavas
lavas init
```
lavas的命令行工具提供了挺多的步骤流程来配置工程，还都是中文，这点很舒服

## 概念介绍

我们这里谈论的是appshell模板

lavas并没有引入过多的抽象概念，基本类似一个普通的vue单页应用脚手架，不过提供了一套通用的静态文件缓存/更新的方案，使我们不需要任何配置就可以实现基本的静态文件缓存功能，这点很赞

### AppShell

这是谷歌提出来的一个[概念](https://developers.google.cn/web/fundamentals/architecture/app-shell)，而lavas的解释如下

> App Shell 架构是构建 PWA 应用的一种方式，它通常提供了一个最基本的 Web App 框架，包括应用的头部、底部、菜单栏等结构。顾名思义，我们可以把它理解成应用的一个「空壳」，这个「空壳」仅包含页面框架所需的最基本的 HTML 片段，CSS 和 javaScript，这样一来，用户重复打开应用时就能迅速地看到 Web App 的基本界面，只需要从网络中请求、加载必要的内容。我们使用 Service Worker 对 App Shell 做离线缓存，以便它可以在离线时正常展现，达到类似 Native App 的体验。

在lavas工程里，它体现为页面的顶部导航、底部导航和侧边栏等组件以及配套的一系列vuex的action和mutation，还有路由中的一些前进/后退的处理逻辑，使我们可以像Native App一样不需要关心细节就可以很简单地修改它们的状态来适应自己的具体需求

### ServiceWorker

这也是谷歌的PWA框架中提出来的[概念](https://developers.google.cn/web/fundamentals/getting-started/primers/service-workers)，lavas的解释如下

> Service Worker 是用 JavaScript 编写的 JS 文件，能够代理请求，并且能够操作浏览器缓存，通过将缓存的内容直接返回，让请求能够瞬间完成。开发者可以预存储关键文件，可以淘汰过期的文件等等，给用户提供可靠的体验。

可以说pwa能做到缓存离线静态文件或其他离线内容就是靠它来控制的，然而处理不好你可能会碰上复杂的缓存不能及时更新的问题，lavas提供了通用的方案来帮助我们缓存static目录内的文件和实现更新功能，在简单情况下我们甚至不需要去在意这个概念就可以开发出pwa应用

### manifest.json

都是谷歌提出的[概念](https://developers.google.cn/web/fundamentals/getting-started/codelabs/your-first-pwapp/#_30)

> 网络应用清单是一个简单的 JSON 文件，使您（开发者）能够控制在用户可能看到应用的区域（例如手机主屏幕）中如何向用户显示应用，指示用户可以启动哪些功能，更重要的是说明启动方法。
>
> 利用网络应用清单，您的网络应用可以：
> 
> * 在用户的 Android 主屏幕进行丰富的呈现
> * 在没有网址栏的 Android 设备上以全屏模式启动
> * 控制屏幕方向以获得最佳查看效果
> * 定义网站的“启动画面”启动体验和主题颜色
> * 追踪您是从主屏幕还是从网址栏启动

使用它才能实现在手机上添加主屏图标和自定义启动页的功能，lavas已提供了默认文件，在此基础上修改就行

### vuetifyjs

lavas使用的UI组件库是遵循MD设计的[vuetifyjs库](https://vuetifyjs.com/vuetify/quick-start)，并非国内常用的框架库，接入可能需要一点理解成本

## 一些问题 

### 没有集中的配置文件

一部分主题配置在config目录中，路由配置在src中，标题和页面主题色等常用配置居然是写死在index.html里的，这点就像一个普通的spa脚手架，缺少一个地方来设置一些常用配置

### vuetifyjs

正在频繁更新，还没出1.0版，谨慎使用，不过文档还算完善

## 总结

lavas确实能使我们不需要在意底层的细节就可以快速开发出pwa应用，当然要深入的话还是要去了解相关知识点的，百度在这一点做得很好，官方文档不仅包括lavas的，还包括整个pwa的介绍，十分全面

唯一奇怪的是github星数居然不到200，怕是没怎么宣传，不过看到有子模块还在频繁更新，我们可以期待未来的发展
