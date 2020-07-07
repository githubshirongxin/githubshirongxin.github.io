---
layout: post
title: k8s node
---

https://www.bilibili.com/video/BV1DJ411N7jU?from=search&seid=8048782610840053535

systemctl enable docker.service

## 安装docker
## 安装k8s
找到阿里镜像站 找k8s的镜像
[Index of /kubernetes/yum/repos/kubernetes-el7-x86_64/](https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/)



### 1，镜像库
**master,node1,node2:**
cd /etc/yum.repos.d
vi Ali-k8s.repo
> [aliyun.k8s]
> name=aliyun.k8s
> baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
> enalbed=1
> gpgcheck=0

scp Ali-k8s.repo root@centos2:/etc/yum.repos.d
scp Ali-k8s.repo root@centos3:/etc/yum.repos.d

### 2，安装kubelet，kubeadm ，kubectl
**master,node1,node2:**
```
yum install kubelet-1.18.1 kubeadm-1.18.1 kubectl-1.18.1 --disableexcludes=kubernetns -y 
```
**[QA]:安装的是哪个版本？**


第一步如果gpgcheck=0没有设置的话，这里安装会出现gpgcheck错误。也有别的办法，当终究还是Ali-k8s.repo里设置不做这个检查比较容易。

### 3，升级内核
**master,node1,node2:**
升级内核 参考：[centos升级内核](https://www.cnblogs.com/xzkzzz/p/9627658.html)
```
uname -r // 查看内核，k8s需要升到5版本以上
```
+ 导入ELRepo仓库的公共密钥
www.elrepo.org //红帽子的

`rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org`

+ 安装ELRepo仓库的yum源
`rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm`

+ 查看可用的系统内核包
`yum --disablerepo="*" --enablerepo="elrepo-kernel" list available`

+ 安装最新版本内核
`yum --enablerepo=elrepo-kernel install kernel-ml`

+ 内核安装好后，需要设置为默认启动选项并重启后才会生效
`sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg`

+ 通过 grub2-set-default 0 命令设置
`grub2-set-default 0`  

+ 生成 grub 配置文件并重启
`grub2-mkconfig -o /boot/grub2/grub.cfg`
`reboot`

+ 验证
  `uname -r`

+ 删除旧内核（可选）
  `rpm -qa | grep kernel`
  `yum install yum-utils // `
  `yum-utils package-cleanup --oldkernels`

### 4，设置k8s之外的系统参数
**master,node1,node2:**
[参考：k8s安装-中](https://www.bilibili.com/video/BV11J411N7RG/?spm_id_from=trigger_reload)

#### 4.1 确认mac地址不同
**master,node1,node2:**
`ifconfig` //三台机器的mac地址最后两位不同，OK
![3.105](/images/2020-06-28-15-51-48.png)
![3.106](/images/2020-06-28-15-53-44.png)
![3.107](/images/2020-06-28-15-54-54.png)

#### 4.2 检查三台主机的UUID必须不同
**master,node1,node2:**
`cat /sys/class/dmi/id/product_uuid`
3.105:`2ce72c31-d4a0-2648-878a-d1a289e24a9a`
3.106:`1E799DE2-54D5-A743-BC38-462068B70CE2`
3.107:`12FBD77E-4AB0-DA45-A0BD-B0144D16DE0B`
各不相同，OK

#### 4.3 关闭swap分区
**master,node1,node2:**
`free -m` //查看swap分区
`swapoff -a`
`cp /etc/fstab /etc/fstab_bak`
`vim /etc/fstab` //注释掉swap那一行
`vi /etc/sysctl.conf` // 增加一行内容
> vm.swappiness = 0
`sysctl -p`
`free -m` //确认swap那行都是0

#### 4.4 关闭防火墙
**master,node1,node2:**
`systemctl disable firewalld.service && systemctl stop firewalld.service`

#### 4.5 启动docker时，避免网桥流量有提示错误，修改下面两项
**master,node1,node2:**
`systemctl start docker.service`
`docker info ` //可能会提示两个警告
> WARNING: bridge-nf-call-iptables is disabled
> WARNING: bridge-nf-call-ip6tables is disabled
`cat /proc/sys/net/bridge/bridge-nf-call-iptables`
`vi /etc/sysctl.conf`
> net.bridge.bridge-nf-call-iptables = 1
> net.bridge.bridge-nf-call-ip6tables = 1
`sysctl -p`
//确认
`systemctl restart docker.service` 
`docker info`//应该没有警告了。
`systemctl enable docker.service` //开机自动启动

#### 4.6 selinux关掉
**master,node1,node2:**
`vim /etc/selinux/config`
> enforcing 改成 disable
`setenforce 0` //重启之前暂时关闭一下
`reboot`

### 5，启动kubelet

#### 5.1 启动kubelet
**master,node1,node2:**
`systemctl enable kubelet.service`
`systemctl start kubelet.service`
//下载k8s所需组件
`kubeadm config images pull` //**会超时！**
> failed to pull image "k8s.gcr.io/kube-apiserver:v1.18.5": output: Error response from daemon: Get https://k8s.gcr.io/v2/: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)

上记超时的解决方案：
`kubeadm version` // 1.18.5 
>kubeadm version: &version.Info{Major:"1", Minor:"18", GitVersion:"v1.18.5", GitCommit:"e6503f8d8f769ace2f338794c914a96fc335df0f", GitTreeState:"clean", BuildDate:"2020-06-26T03:45:16Z", GoVersion:"go1.13.9", Compiler:"gc", Platform:"linux/amd64"}

`kubeadm config images list --kubernetes-version=v1.18.5`
>W0628 17:10:40.571146    8582 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
k8s.gcr.io/kube-apiserver:v1.18.5
k8s.gcr.io/kube-controller-manager:v1.18.5
k8s.gcr.io/kube-scheduler:v1.18.5
k8s.gcr.io/kube-proxy:v1.18.5
k8s.gcr.io/pause:3.2
k8s.gcr.io/etcd:3.4.3-0
k8s.gcr.io/coredns:1.6.7

解决上记问题，列出各个需要的组件，然后用国内的镜像来拉下来docker镜像
`docker search mirrorgooglecontainers`
`docker pull mirrorgooglecontainers/kube-apiserver:v1.18.5`
`docker pull mirrorgooglecontainers/kube-controller-manager:v1.18.5 `
`docker pull mirrorgooglecontainers/kube-scheduler:v1.18.5 `
`docker pull mirrorgooglecontainers/kube-proxy:v1.18.5`
`docker pull mirrorgooglecontainers/pause:3.2`
`docker pull mirrorgooglecontainers/etcd:3.4.3-0` `docker pull coredns/coredns:1.6.7`

 注意：最后一个的库名字中国让用，就是coredns/
 https://www.bilibili.com/video/BV11J411N7RG/?spm_id_from=trigger_reload 22：26处 停留

 之后再看https://www.jianshu.com/p/f4ac7f4555d3

[课题]3.105上kubeadm version时1.18.5,3.105已经安装了