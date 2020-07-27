---
layout: post
title: vuepress gitlab local
---

从gitlab的sample page中找到vuepress 
https://gitlab.com/pages/vuepress

这里的工程.gitlab-ci.yml已经配置好了。
里面yarn install 安装所有依赖包。

安装哪些呢？

package.json里面的所有依赖都写到里面了。
当前只有一条就是vuepress
 ![](/images/2020-07-23-22-21-05.png)
 

[.gitlab-ci.yml配置参考](https://www.jianshu.com/p/3bb437b4edb9)


然后clone到本地

## 配置本地服务器

改改之后，上传到gitlab本地服务器

gitlab本地服务器，启动gitlab pages

为本地服务器配置一个编译环境 Runner

## 写文章
在docs/下写文章md文件

git push到本地服务器

自动触发Running对静态文本编译成浏览器可以识别的html，js,css.

发布到什么地方，然后随着gitlab服务器一起启动了起来，能访问gitlab本地服务器。就能访问该blog

## blog的分类
base = 库名
username.gitlab.io/reponame // 这种是project级别的，所以被称作project blog
base = /
username.gitlab.io  //这种是账号级别的，所以被称作account blog
orgnazition.gitlab.io // 这种是组织级别的，所以被称作公司 blog。

