---
layout: post
title: k8s 
---

## 解决了什么问题

## install

### STEP1，所有机器都安装docker
略
把某用户放到docker权限来
```
sudo groupadd docker
sudo usermod -aG docker $USER
```
该用户重新开个终端，就有docker权限了。

修改加速器/etc/docker/daemon.json
重启docker

### STEP2，安装K8s 的准备工作
1. 配置k8s的国内源
   创建 /etc/apt/sources.list.d/kubernetes.list
2. 添加 `chomd 666 该文件`
    ![](/images/2020-06-26-13-46-33.png)
3. 执行 `sudo apt update`
   3.1 报Error ，gpg --keyserver 
    ![](/images/2020-06-26-13-48-41.png)

4. 再次 `sudo apt update`更新系统下载源数据列表
5. 禁止基础设施
   + 禁止防火墙 `sudo ufw disable`
   + 关闭swap `sudo awapoff -a` 
   + 永久关闭swap ` sudo sed -i 's/.*swap.*/#&/' /etc/fstab`
   + 禁止selinux
  ```bash
  sudo apt install -y selinux-utils
  setenforce 0
  shutdown -r now
  sudo getenforce
  ```
### STEP3. 安装k8s
