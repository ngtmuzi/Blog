---
title: TypeScript 学习笔记（后端）
desc: 凡是能用js实现的，都应该用ts……
author: ngtmuzi
category: 班门弄斧
date: 2020-10-06 13:22:22
tags:
  - TypeScript
  - TS
---

说来惭愧，我是在换了工作之后才考虑在项目中完整使用 TypeScript（后简称 TS）的，因此接触时间并不算很长，加上网上介绍 TS 的文章应该是多如牛毛了，在这里总结一下自己的开发和学习经验，摘抄一些文档内容，权当记录参考。

## Why TS?

- 为 JS 补充上静态类型定义系统
  - 在编码时就能发现类型和语法错误
  - 更好的编辑器开发支持（尤其是 vscode）
- 提供最新的 JS 特性支持（类似 babel）
- 迎合 deno（？）

## JS 迁移到 TS

### 语法上的迁移

单文件的 JS 到 TS 实际是很简单的，改扩展名，然后有问题解决问题即可，一般会碰到的常见问题会有：

#### `require`引用的模块没有类型定义

尝试查找有没有官方定义文件`npm i @types/xxx -D`，没有的话可以自己在`d.ts`内简单定义`declare module xxx;`，或者自己为其写一个完整的类型定义

#### 变量/参数缺乏类型定义

这就是应该手动补充的内容，如果结构较为复杂或者暂时还对 TS 掌握不够好，可以先简单用 `any` 占位

### 项目上的迁移

我一开始也在这个问题上纠结很久，总感觉项目中做部分迁移的话，会让其他开发人员困扰，但实际只要调整好`tsconfig.json`配置文件（比如将 JS 输出目录设为原目录，直接替换原文件），我们是可以做到不改动其他 JS 来逐步迁移的，最终执行的也还是 JS 文件，整体上不会有什么变化

## 开发技巧

比较简单的用法可以直接看官方文档，就不赘述了，下面说一些日常开发总结的

### 善用工具类型

| 语法                           | 简述&示例                                                                                                                     |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `Partial<Type>`                | 将`Type`内所有属性改为可选的类型 <br> `Partial<{a:string}>` => `{a?:string}`                                                  |
| `Required<Type>`               | 与 Partial 相反，将`Type`内所有属性改为必须类型 <br> `Partial<{a:string}>` => `{a?:string}`                                   |
| `Readonly<Type>`               | 将`Type`内所有属性改为只读的类型 <br> `Readonly<{a:string}>` => `{readonly a:string}`                                         |
| `Record<Keys,Type>`            | 构建一个键类型为`Keys`，值类型为`Type`的类型 <br> `Record<'a' 丨 'b',string>` => `{a:string,b:string}`                        |
| `Pick<Type, Keys>`             | 根据`Keys`键列表构建`Type`的子集类型 <br> `Pick<{a:string,b:string}, 'a'>` => `{a:string}`                                    |
| `Omit<Type, Keys>`             | 排除`Keys`键列表构建`Type`的子集类型 <br> `Omit<{a:string,b:string}, 'a'>` => `{b:string}`                                    |
| `Exclude<Type, ExcludedUnion>` | 对联合类型`Type`做排除计算 <br> `Exclude<'a' 丨 'b' 丨 'c', 'a'>` => `'b' 丨 'c'`                                             |
| `Extract<Type, Union>`         | 对联合类型`Type`做交集计算 <br> `Extract<'a' 丨 'b' 丨 'c', 'a' 丨 'd'>` => `'a'`                                             |
| `NonNullable<Type>`            | 对联合类型`Type`排除空类型(`null`和`undefined`)，但实际上还是能赋空值 <br> `NonNullable<string 丨 null>` => `string`          |
| `Parameters<Type>`             | 返回函数类型`Type`的参数类型元组 <br> `Parameters<(a: string, b: any) => any>` => `[a: string, b: any]`                       |
| `ConstructorParameters<Type>`  | 返回类型`Type`的构造函数的参数类型元组 <br> `ConstructorParameters<typeof Date>` => `[value: string 丨 number 丨 Date]`       |
| `ReturnType<Type>`             | 返回函数类型`Type`的返回值类型 <br> `ReturnType<typeof Number>` => `number`                                                   |
| `InstanceType<Type>`           | 返回构造函数类型`Type`的实例类型 <br> `InstanceType<typeof Number>` => `Number`                                               |
| `ThisParameterType<Type>`      | 返回函数类型`Type`的上下文`this`的类型，若未指定则返回`unknown`<br> `ThisParameterType<(this: string) => number>` => `string` |
| `OmitThisParameter<Type>`      | 返回不包含`this`参数的`Type`函数类型<br> `OmitThisParameter<(this: string) => number>` => `() => number`                      |

### 容易混淆的点

#### 值和类型没有区分清楚

这是一个比较常见的错误，我觉得还是要靠多写自己去总结，总结如下

