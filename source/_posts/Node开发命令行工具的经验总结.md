---
title: Node开发命令行工具的经验总结
desc: 入门之后还是很方便的
author: ngtmuzi
category: 班门弄斧
date: 2019-02-27 14:23:02
tags: 
- nodejs
- shell
- bash
---

最近一直没怎么写博客，主要是工作中没太多产出，更多的可能还是偷懒…

## 背景

近半年接手了一个非web开发类的工作，一直跟数据、数据库和脚本打交道，原项目是windows服bat脚本和.NET命令行程序来跑各种任务的。之前我没有太接触过shell这块，碰到这些bat脚本确实有点把我恶心到了，各方面相比bash来说还是有很大差距，于是我着手开始做迁移工作

这里推荐一个简单的bash入门教程
> [bash-handbook: 谨以此文档献给那些想学习Bash又无需钻研太深的人。](https://github.com/denysdovhan/bash-handbook/blob/master/translations/zh-CN/README.md)

而复杂逻辑的exe部分，还是用我熟悉的node.js来重构，在此之前我还没有过命令行的开发经验，算是摸着石头过河

### 参考资料

> [Node.js 命令行程序开发教程-阮一峰](http://www.ruanyifeng.com/blog/2015/05/command-line-with-node.html)

## 开始

### 获取和解析参数

### 从输入流读入数据

### 注意当前目录

### 链接为全局指令