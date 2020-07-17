---
layout: post
title: 【高可用】nginx + keepalive 实现主从切换
---
keepalive 两台机器都装
一台设置master，一台设置backup
写一个脚本放到keepalive里，监控nginx进程，进程不见了，就重启nginx，重启失败就停止本机keepalive进程，会自动IP漂移到backup。master恢复后会还原。

https://juejin.im/post/5d9066e06fb9a04e2c014e3d

作者写的很好。看他的就行。

