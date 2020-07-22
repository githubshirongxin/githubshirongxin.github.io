---
layout: post
title: 【webpack】npm,node,webpack,vue关系梳理
---

## npm与node
npm和node是绑定在一起的可执行文件。
但npm作者和node作者并不是一个。
当初npm写完代码之后，想和jqurey联合，但是jquery作者没看上。
node作者正缺少包管理工具，于是和npm一拍即合。node火了以后，npm就内嵌到node中了。

如今想单独安装npm已经不可能了，单并不表示npm就需要node。npm是个独立的软件。
从使用上看npm也没有和node有太大关系，它仍旧做的是包管理。

## node与javasript
node是后台框架。他是javascript，以前固定思维：js就是前台。现在不是了，借助谷歌的
jsV8引擎，可以高性能地解析js做后端应用，什么router，后台逻辑，启动http服务，写处理逻辑，连接数据库，它都能做。这就让以前的前端程序员很快地掌握后端，就可以做全栈了。
### 问题1：node做的后端有没有局限性？使用场景是什么？
node处理复杂业务和大数据存储不擅长。

### 问题2：基于node框架应该都是后端框架了吧，有哪些流行的框架？业界真正使用的。


### 问题3：如果node框架都是后台框架，那么前台框架用什么？

## nodejs的调用方式看node是后台还是前台
node用js文件，模块化调用，分别开发。然后用node xxx.js运行。
非常像java的方式。java xxx.jar
能启动http，能操作fs，能buffer、stream、router后台的功能几乎都可以做。

## node的后端有什么优势
非阻塞式io，让node可以变成高性能处理而不依赖内存。
使用更少的服务器处理更高的并发。
但它适合上传下载都不是很大数据的场合，why？
只适合少量数据，大量并发的场合。

## webpack与node
webpack与node没有关系
webpack的组织方式使用了npm而已。
而npm和node绑定安装了而已，就是node名气大了，npm成了node的附庸，实际上大家只想用npm不想用node。

## webpack与npm功能有什么不同
npm解决的是js依赖包关系，和maven、gradle解决的是java的依赖关系，自动下载。
webpack解决的是把scss和es6，vue等编译成浏览器认识的css，html，js。还有写扩展功能plugin实现的。

## webpack与npm
### webpack是npm生态中的一环。npm的库里有webpack这样的模块。
`npm install D webpack`本地安装之后，会在当前目录下的node_modules/下面安装webpack

### 完全可以单独运行`webpack `，它也不是非得需要配置文件才行。
把配置的内容写到命令行里一样可以。配置文件就是为了方便。
webpack的配置文件也不是npm install之后就自动创建好，而是需要自己创建，自己一行行写。
这和`npm init -y`就创建package.json不同。

### npm script可以调用webpack
当然npm的script也就是package.json中<script>段。调用shell，当然可以调用webpack了。
`xxx : "webpack ..."`
然后命令行npm run xxx 这没有什么。npm script能做的也很少，属于很原始很初级的调用。
写个shell也能这样调用。

## vue和webpack
vue内置了webpack

## vue与node
vue与node没有关系

## vue是前台框架吗
不是。vue就是个库。vue生态可以做前台框架。得很多何在一起才是个框架。




