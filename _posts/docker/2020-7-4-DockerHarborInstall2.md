---
layout: post
title: 【docker】docker本地库安装，harbor安装（后续改善）
---

修改docker卷位置
首先把公司分给我的raid盘挂载过来。
然后把harbor默认的dockers卷修改在这块磁盘上。这样至少存储上提高了点安全性。

1，作为samba的客户端挂载服务端的磁盘
[samba实现分布式磁盘](https://githubshirongxin.github.io/Smba%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%A3%81%E7%9B%98/)

已经挂完 /data_cjb/

- 确认一下挂载情况：
```
mount | grep data_cjb
或者
df -hT | grep data_cjb
或者
cat /etc/fstab
```

2，修改harbor/docker-compose.yml , harbor.yml
- harbor.yml
```
 # Log configurations
   location: /var/log/harbor # 不改了，这块丢就丢了。
 # The default data volume
 #  data_volume: /data #这得改成mount好的raid盘目录
 data_volume: /data_cjb/harbordata
```

-docker-compose.yml

所有/data/ 都改成/data_cjb/harbordata/