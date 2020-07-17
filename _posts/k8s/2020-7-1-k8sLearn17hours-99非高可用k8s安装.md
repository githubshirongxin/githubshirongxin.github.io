---
layout: post
title: 【k8s】【视频教程70P】（一，非高可用k8s安装）
---

最后讲了高可用部署。

似还不错。先记下。以后再追记貌

https://www.bilibili.com/video/BV1ME411g7EU?p=10


最后一节构建了高可用的k8s

Harbor--》k8s--》软路由（koolshare）科学上网用。
![](/images/2020-07-16-15-58-02.png)

## 环境介绍
- master 66.10
- node1 66.20
- node2 66.21
- harbor 
> 都是centos7 （最好最稳定是4.4内核以上）
- koolshare（koolss） ：192.168.1.1 -》 192.168.66.1
> window虚拟机BIOS模式,PE模式，IDE虚拟磁盘类型。
- 基于最新稳定版Kubernetes 1.15.1（2019年8月发布）进行构建及讲解

###  centos 4个节点（master，node1，2，harbor）
// 略 网络：仅仅主机

###  软路由 （为了所有节点科学上网）
// windows10 虚拟机，PE
https://www.bilibili.com/video/BV1ME411g7EU?p=11
- 光盘换成koolshare20140119，里面有"IMG写盘工具.exe"
- 添加网卡：nat模式
- 网页配置ssr

###  harbor 简单的构建 
// 略

---

## 部署k8s 
## 一， 【所有节点】部署前环境准备
https://www.bilibili.com/video/BV1ME411g7EU?p=12

#### 设置系统主机名
1. 修改主机名
`hostnamectl set-hostname k8s-master-01`
`hostnamectl set-hostname k8s-node-01`
`hostnamectl set-hostname k8s-node-02`
2. 加入/etc/hosts
小环境不建议用DNS，DNS容易有单点故障，造成k8s不可用
所以，可以直接修改/etc/hosts
3. 拷贝/etc/hosts 到三台节点

#### 安装依赖包
`yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat libseccomp wget vim net-tools git`


####  关闭防火墙 为iptables并设置空规则
```bash
systemctl stop firewalld && systemctl disable firewalld 
yum -y install iptables-services && systemctl start iptables && systemctl enable iptables && iptables -F && service iptables save 
```

#### 关闭SELINUX
```bash
swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
```

#### 针对k8s调整内核参数

//三条是必须，其他是优化

```bash
cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1 #必须
net.bridge.bridge-nf-call-ip6tables=1 #必须
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0 #禁止使用swap空间
vm.overcommit_memory=1 #不检查物理内存是否够用
vm.panic_on_oom=0 #开启OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1 #关闭ipv6协议
net.netfilter.nf_conntrack_max=2310720
EOF

cp kubernetes.conf /etc/sysctl.d/kubernetes.conf
sysctl -p /etc/sysctl.d/kubernetes.conf
```

#### 调整系统时区
```bash
#设置系统时区为中国上海
timedatectl set-timezone Asia/Shanghai
#将当天的UTC事件写入硬件时钟
timedatectl set-local-rtc 0
#重启跟系统时间有关的两个服务，否则他们不起作用
systemctl restart rsyslog
systemctl restart crond
```

#### 关闭不需要的服务
`systemctl stop postfix && systemctl disable postfix`


#### 设置log rsyslogd和systemd journald
不转发到syslog减轻服务器压力
```bash
mkdir /var/log/journal # 持久化保持日志的目录

mkdir /etc/systemd/journald.conf.d

cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
# 持久化保持到磁盘
Storage=persistent

#压缩历史日志
Compress=yes

SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000

# 最大占用空间 10G
SystemMaxUse=10G

# 单日志文件最大200M
SystemMaxFileSize=200M

# 日志保存时间2周
MaxRetentionSec=2week

# 不讲日志转发到syslog
ForwardToSyslog=no
EOF

systemctl restart systemd-journald

```

#### 升级系统内核为4.44
3.1版本有小bug，导致docker运行不稳定
```bash
下载内核源
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

安装最新版本内核
yum --enablerepo=elrepo-kernel install -y kernel-lt

查看可用内核
cat /boot/grub2/grub.cfg |grep menuentry

设置开机从新内核启动
grub2-set-default "CentOS Linux (4.4.221-1.el7.elrepo.x86_64) 7 (Core)"

查看内核启动项
grub2-editenv list

重启系统使内核生效
reboot

查看内核版本是否生效
uname -r

```

#### kube-proxy开启ipvs的前置条件
增加访问效率

```bash
modprobe br_netfilter

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- ip_conntrack_ipv4
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash

/etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```

#### 安装Docker软件
```bash
yum install -y yum-utils  device-mapper-persistent-data  lvm2

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

yum update -y && yum install -y docker-ce docker-ce-cli containerd.io

### 创建/etc/docker目录
mkdir /etc/docker

### 写配置文件daemon.json
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts":["native.cgroupdirver=systemd"],
  "log-driver":"json-file",
  "log-opts":{
      "max-size": "100m"
  }

}
EOF

### 存放docker的配置文件
mkdir -p /etc/systemd/system/docker.service.d

### 重启docker并设置开机启动
systemctl daemon-reload && systemctl restart docker.service && systemctl enable docker.service
```

#### 提示：做虚拟机快照，备份好
【Tip】手顺找一份。

### 安装kubeadm

+ 每个节点都要安装

[可以参考《Kubeadm 部署安装》99%相同](https://blog.csdn.net/dgqg1223/article/details/106535223/)



```bash
// 导入阿里云的yum仓库
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

//安装三个服务
yum install -y kubeadm-1.15.1 kubectl-1.15.1 kubelet-1.15.1

// 必须开机自启
systemctl enable kubelet.service
```
<font face="verdana" color=blue size=5>所有节点 “共通” 操作：
+ 调整系统设置
+ 安装docker
+ 安装命令：kubeadm、kubectl、kubelete

</font>
---

## 二，【主节点】初始化主节点

**初始化之前注意：**
kubeadm init命令需要访问到google，如果不能科学上网的。只能先下载到kubeadm-basic.images.tar.gz （多个镜像的集合），然后传到主节点。然后再执行kubeadm init！

#### 首先，（不能科学上网的话）下载镜像，kubeadm init所依赖的镜像。然后导入到主节点的docker images里。
+ 这步可以省略

##### k8s 部署需要访问google k8s云 ，若无法访问google ，上传所需镜像包即可
###### kubeadm-basic.images.tar.gz

```bash
kubeadm-basic.images.tar.gz winscp上传到主节点。

# 解压
tar -zxvf kubeadm-basic.images.tar.gz

## 编写并运行脚本导入到docker
#!/bin/bash

ls kubeadm-basic.images > /tmp/image-list.txt

cd /root/kubeadm-basic.images

for i in $( cat /tmp/image-list.txt )
do
        docker load -i $i
done

rm -rf /tmp/image-list.txt
```

#### 【主节点】初始化

```bash
# 生成 kubeadm 默认初始化模板
kubeadm config print init-defaults > kubeadm-config.yaml
```

修改kubeadm-config.yaml
`vim kubeadm-config.yaml`

```bash
localAPIEndpoint:
# 修改kubeadm-config.yaml 中advertiseAddress为当前节点的ip地址 
# 修改kubernetesVersion 版本
advertiseAddress: 192.168.3.105
kubernetesVersion: v1.15.1

networking:
# 添加 podSubnet: "10.244.0.0/16" flannel插件网段
podSubnet: "10.244.0.0/16" #←this
serviceSubnet: 10.96.0.0/12

## 在最后插入下列配置 将默认调度模式改为ipvs模式
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs
```


```bash
# 初始化代码 （--experimental-upload-certs 自动颁发证书）
kubeadm init --config=kubeadm-config.yaml --experimental-upload-certs | tee kubeadm-init.log
```

```bash
[init] Using Kubernetes version: vX.Y.Z
[preflight] Running pre-flight checks
[kubeadm] WARNING: starting in 1.8, tokens expire after 24 hours by default (if you require a non-expiring token use --token-ttl 0)
// 省略一部分

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the addon options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

##### log能看出来
- /etc/kubernetes/pki 存放的都是kubernetes各个组件之间通讯需要的私钥和证书，这些都是init命令生成的。
- DNS名称，地址
- /etc/kubernetes/存放所有配置文件
  

##### 【主节点】 后续操作
首先，执行如下命令，完成配置的初始化：
```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
这样之后，kubectl就可以用了，尝试一下

```bash
kubectl get node
```
只有master-01，**NotReady**状态（因为还没有flannel网络）

##### 【主节点】安装网络插件
K8S的网络插件有很多种类可供选择，官网链接中已经给出了各种插件的安装方法。
这里用Weave Net
```bash
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
或者
```bash
mkdir -p install-k8s/core
mv kubeadm-init.log kubeadm-config.yaml install-k8s/core
mkdir -p install-k8s/plugin/flannel
cd install-k8s/plugin/flannel

wget https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
kubectl create -f kube-flannel.yml

//前两个命令可以合并成一条
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.9.1/Documentation/kube-flannel.yml
```

网络插件安装完后，来确认一下服务是否启动成功。 
-w就是监控
```bash
$ kubectl get pod -n kube-system -o wide -w 
NAME                                              READY   STATUS    RESTARTS   AGE
coredns-576cbf47c7-dch4w                          1/1     Running   1          90m
coredns-576cbf47c7-whnzs                          1/1     Running   0          90m
etcd-izm5e9951st9peq42t8fkxz                      1/1     Running   0          89m
kube-apiserver-izm5e9951st9peq42t8fkxz            1/1     Running   0          89m
kube-controller-manager-izm5e9951st9peq42t8fkxz   1/1     Running   0          89m
kube-proxy-s5khx                                  1/1     Running   0          90m
kube-scheduler-izm5e9951st9peq42t8fkxz            1/1     Running   0          89m
weave-net-29mjv                                   2/2     Running   0          75m
```

如果都是Running状态就说明启动起来了（刚安装完网络插件可能要等一会）

`ifconfig`
会多一个flannel

`$ kubectl get nodes`
master节点就**Ready**了

#### 消除master节点的隔离（可选）

默认情况下，k8s是不会在master节点上自动部署业务上需要的应用的，如果是测试环境机器数量比较少，可以将这个隔离给去掉（这样master节点也可以部署应用了）。
```bash
$ kubectl taint nodes --all node-role.kubernetes.io/master-
```
会看到类似这样的输出
```bash
node/izm5e9951st9peq42t8fkxz untainted
error: taint "node-role.kubernetes.io/master:" not found
```
说明已经消除了隔离

<font face="verdana" color=blue size=5>主节点：init、创建flannel网络</font>

---

## 三，【所有子节点】创建worker节点
（memo一下备用）一会要创建worker节点的时候需要用这句命令来让worker加入集群。
```bash
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```
执行完后，在子节点上会初始化pod，所以需要稍微等等

另外还有这个命令，查看k8s集群中的所有节点状态
```bash
$ kubectl get pod -n kube-system -o wide -w 
```
一会，所有pod都变成Running了

然后随便哪个节点上
`kubectl get pod`
可以看到所有节点**都是Ready**

<font face="verdana" color=blue size=5>子节点：只需要执行join就可以了</font>


## 四，后续打扫（可选）
install-k8s/重要目录备份到/usr/local
删除不用的安装包


## 高可用k8s
https://www.bilibili.com/video/BV1ME411g7EU?p=68
