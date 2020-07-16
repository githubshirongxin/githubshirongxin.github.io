---
layout: post
title: 【k8s】k8s集群安装 centos7 k8s v1.15.3（3.105安装方法，验证好用）
---

我机器192.168.3.105安装的就是这个方法。
验证过好用。

参考：https://blog.csdn.net/mtldswz312/article/details/98732198?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase

## 第一章 

### 1.1 前期准备工作：
##### （1）关闭防火墙，和selinux
```
yum -y install wget vim net-tools ntpdate
systemctl stop firewalld
systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
systemctl stop NetworkManager
systemctl disable NetworkManager
```
##### （2）时钟同步
```
echo '*/10 * * * * /usr/sbin/ntpdate -s 10.100.60.6 >/dev/null 2>&1 && /sbin/clock -w' > /var/spool/cron/root 
service crond restart 
ntpdate -s 10.100.60.6
```
##### （3）私有主机禁用swap分区
```
swapoff -a   
vi /etc/fstab
[root@master01 ~]# cat /etc/fstab 
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=8d103c59-0306-4493-94f2-1e3726d87cfb /boot                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```
##### （4）互相解析
```
cat >> /etc/hosts << EOF
192.168.3.105 centos1
192.168.3.106 centos2
192.168.3.107 centos3
192.168.3.105 master01
192.168.3.106 node01
192.168.3.107 node02
EOF
```
##### （5）master对node节点ssh互信
```
[root@master01 ~]# ssh-keygen
[root@master01 ~]# ssh-copy-id node01
[root@master01 ~]# ssh-copy-id node02
```
##### （6）修改内核参数
```
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```


## 第二章 

注意：以下在所有节点执行（master+node），安装docker，kubeadm，kubelet
##### 1、配置docker源
```
cat >> /etc/yum.repos.d/docker.repo <<EOF
[docker-repo]
name=Docker Repository
baseurl=http://mirrors.aliyun.com/docker-engine/yum/repo/main/centos/7
enabled=1
gpgcheck=0
EOF

#配置kubernetes源
cat >> /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF

yum clean all
yum makecache
```

##### 2、安装kubeadm和相关工具包（所有节点）
```
yum install -y docker --disableexcludes=docker-repo
systemctl enable docker && systemctl start docker

yum install -y kubelet-1.15.3 kubeadm-1.15.3 kubectl-1.15.3 --disableexcludes=kubernetes
systemctl enable kubelet && systemctl start kubelet
```
（现在版本1.18.5国内镜像没有，所以降低了一点）

##### 3、初始kubeadm集群环境（仅master节点）
```
kubeadm init --image-repository=registry.aliyuncs.com/google_containers --service-cidr=192.168.0.0/16 --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.15.3
```

**安装完成后记录一下**
```
[init] Using Kubernetes version: v1.15.3
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.3.105:6443 --token p9916m.96bm9res6l15zusq \
    --discovery-token-ca-cert-hash sha256:3ce5cc691f042b2ee466365064fa858132e8149ca8e623bd6d2997ef0601c886
[root@centos1 ~]#
 
```

**执行操作：**
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

[root@master01 ~]# kubectl get nodes
NAME       STATUS     ROLES    AGE     VERSION     
master01   NotReady   master   2m42s   v1.15.3      #状态是Notready，在等待网络的加入

[root@master01 ~]# kubectl get pod -n kube-system      #看到有2个pod处于pending
NAME                               READY   STATUS    RESTARTS   AGE
coredns-bccdc95cf-bhtms            0/1     Pending   0          4m18s
coredns-bccdc95cf-jmbds            0/1     Pending   0          4m17s
etcd-master01                      1/1     Running   0          3m30s
kube-apiserver-master01            1/1     Running   0          3m15s
kube-controller-manager-master01   1/1     Running   0          3m23s
kube-proxy-n62h7                   1/1     Running   0          4m18s
kube-scheduler-master01            1/1     Running   0          3m14s
```

##### 4、在master节点上安装flannel网络
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```
(<font color=red>raw.githubusercontent.com找不到。因为对应的IP被墙了。解决见下面TroubleShooting问题1</font>)

```
[root@master01 ~]# kubectl get pod -n kube-system    #看到所有的pod都处于running状态。
NAME                               READY   STATUS    RESTARTS   AGE
coredns-bccdc95cf-bhtms            1/1     Running   0          6m4s
coredns-bccdc95cf-jmbds            1/1     Running   0          6m3s
etcd-master01                      1/1     Running   0          5m16s
kube-apiserver-master01            1/1     Running   0          5m1s
kube-controller-manager-master01   1/1     Running   0          5m9s
kube-flannel-ds-amd64-6jjwf        1/1     Running   0          59s
kube-proxy-n62h7                   1/1     Running   0          6m4s
kube-scheduler-master01            1/1     Running   0          5m
```

##### 5、添加计算节点（在节点上执行）
```
[root@node01 ~]# kubeadm join 192.168.3.105:6443 --token p9916m.96bm9res6l15zusq \
    --discovery-token-ca-cert-hash sha256:3ce5cc691f042b2ee466365064fa858132e8149ca8e623bd6d2997ef0601c886 

[root@node02 ~]# kubeadm join 192.168.3.105:6443 --token p9916m.96bm9res6l15zusq \
    --discovery-token-ca-cert-hash sha256:3ce5cc691f042b2ee466365064fa858132e8149ca8e623bd6d2997ef0601c886

[root@master01 ~]# kubectl get nodes
NAME       STATUS     ROLES    AGE     VERSION
master01   Ready      master   8m55s   v1.15.3
node01     NotReady   <none>   37s     v1.15.3
node02     NotReady   <none>   14s     v1.15.3
```

##### 6、部署dashboard（在master上操作）
```
[root@master01 ~]# kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta1/aio/deploy/recommended.yaml

[root@master01 ~]# kubectl get pods --namespace=kubernetes-dashboard      #查看创建的namespace
NAME                                          READY   STATUS    RESTARTS   AGE
kubernetes-dashboard-5c8f9556c4-w6pzj         1/1     Running   0          7m46s
kubernetes-metrics-scraper-86456cdd8f-7js7v   1/1     Running   0          7m46s

[root@master01 ~]# kubectl get service --namespace=kubernetes-dashboard   #查看端口映射关系
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.98.83.31     <none>        8000/TCP        55m
kubernetes-dashboard        NodePort    10.107.192.48   <none>        443:30520/TCP   55m
```

##### 7、修改service配置，将type: ClusterIP改成NodePort
```
[root@master01 ~]# kubectl edit service kubernetes-dashboard --namespace=kubernetes-dashboard
如下：
spec:
  clusterIP: 10.107.192.48
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 30924
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort      #注意这行。
```

##### 8、创建dashboard admin-token（仅master上执行）
```
cat >/root/admin-token.yaml<<EOF
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
EOF
```
（<font color=red>直接拷贝会有乱字符,从别的网址搜索admin-token.yaml内容都一样，试试看</font>）

###### 创建用户
```
[root@master01 ~]# kubectl create -f admin-token.yaml
clusterrolebinding.rbac.authorization.k8s.io/admin created
serviceaccount/admin created
```
###### 获取token

```
[root@centos1 ~]# kubectl describe secret/$(kubectl get secret -nkube-system |grep admin|awk '{print $1}') -nkube-s ystem
Name:         admin-token-pq6z6
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin
              kubernetes.io/service-account.uid: 3f9d518a-228a-4489-9df7-a391e0f0fb48

Type:  kubernetes.io/service-account-token

Data
====
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi1wcTZ6NiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjNmOWQ1MThhLTIyOGEtNDQ4OS05ZGY3LWEzOTFlMGYwZmI0OCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTphZG1pbiJ9.rTrAY1oFNbGmcCtJ_uecMzaFgpCuoJoGMYJfmlppDD7DaoaWZjLYCA9GQWFI9yEb0lyu3Y8jotO_MRO8GuPQ5Tv1CiplEaeTGWf77hTM1iIqRFTGV67yZOKaVoyr-Ck-k5UVGwq5uEGSiYNUm18q88qr7CBS9Xjw5x2lrzAo4tucpsr6NeVWL29MBoE6KVb42RiIelCxC_I3zdmYNpv4YWPaT-YOHJCcwx5N8cxcba44pSUEtOIBM8rVHTWdbE9LbdJ6etIewDKxH8RCqdzU4vq7u5oGXoVsNVwyfIiObQSi9b9-J5aiEGZaqj2UmlaiROzytf03GEUgCj-ES8SMfg
ca.crt:     1025 bytes
```

#####  9.登录dashboard 必须用火狐浏览器https://192.168.3.105:30924
选token方式，输入上面的token

![](/images/2020-06-29-21-16-42.png)
---

###### TrubleShooting:

**[问题1：raw.githubusercontent.com 找不到不识别]**
**解决：**
```
https://site.ip138.com/raw.Githubusercontent.com/

输入raw.githubusercontent.com

查询IP地址

修改hosts Ubuntu，CentOS及macOS直接在终端输入

sudo vi /etc/hosts
 

添加以下内容保存即可 （IP地址查询后相应修改，可以ping不同IP的延时 选择最佳IP地址）

# GitHub Start
52.74.223.119 github.com
192.30.253.119 gist.github.com
54.169.195.247 api.github.com
185.199.111.153 assets-cdn.github.com
151.101.76.133 raw.githubusercontent.com
151.101.108.133 user-images.githubusercontent.com
151.101.76.133 gist.githubusercontent.com
151.101.76.133 cloud.githubusercontent.com
151.101.76.133 camo.githubusercontent.com
151.101.76.133 avatars0.githubusercontent.com
151.101.76.133 avatars1.githubusercontent.com
151.101.76.133 avatars2.githubusercontent.com
151.101.76.133 avatars3.githubusercontent.com
151.101.76.133 avatars4.githubusercontent.com
151.101.76.133 avatars5.githubusercontent.com
151.101.76.133 avatars6.githubusercontent.com
151.101.76.133 avatars7.githubusercontent.com
151.101.76.133 avatars8.githubusercontent.com
# GitHub End

```

