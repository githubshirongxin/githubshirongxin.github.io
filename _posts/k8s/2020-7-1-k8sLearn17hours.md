---
layout: post
title: 【k8s】容器 Kubernetes 视频教程全集（70P）(还没看，先memo一下，待续)
---

貌似还不错。先记下。以后再追记

https://www.bilibili.com/video/BV1ME411g7EU?p=10


最后一节构建了高可用的k8s

Harbor--》k8s--》软路由（koolshare）科学上网用。
![](/images/2020-07-16-15-58-02.png)

## 环境
- master
- node1
- node2
> 都是centos7 （最好最稳定是4.4内核以上）
- koolshare（koolss） ：192.168.1.1 -》 192.168.66.1
> window虚拟机BIOS模式,PE模式，IDE虚拟磁盘类型。

基于最新稳定版Kubernetes 1.15.1（2019年8月发布）进行构建及讲解

## 1. 基础环境搭建

###  1.1 centos 4个节点（master，node1，2，harbor）
// 略 网络：仅仅主机

###  1.2 软路由 （为了所有节点科学上网）
// windows10 虚拟机，PE
https://www.bilibili.com/video/BV1ME411g7EU?p=11
- 光盘换成koolshare20140119，里面有"IMG写盘工具.exe"
- 添加网卡：nat模式
- 网页配置ssr

###  1.3 harbor 简单的构建 
// 略

## kubeadm安装方案
+ 进程能够自愈
+ 仍旧能够看到配置文件