---
layout: post
titile: 【k8s】【视频教程70P】(二. harbor安装后k8s节点配置)
---

## harbor安装
略，参考前两篇文章。
[【docker】harbor 单机https安装（成功安装）](https://githubshirongxin.github.io/DockerHarborInstall/)
[【docker】nginx反向代理harbor两节点共享NFS存储、DB，弱可用Harobor部署（后续改善）](https://githubshirongxin.github.io/DockerHarborInstall2/)

## 修改harbor以及k8s各个节点的daemon.json
都加上一句话
```bash
vi /etc/docker/daemon.json
"insecure-registries":["https://harbor.ccbjb.com.cn"]
```
这url是你的harbor的域名。
我这个域名已经加入到公司的域名服务器里了，所以不用修改每天机器的hosts。

因为是自签证书，所以每个客户端dockers配置文件都需要加入此配置
强制承认该URL是安全的。

<font color=red>注意，前面如果有内容，别忘了在上一行后面加逗号！，否则docker启动失败</font>


## 然后重新启动docker
`systemctl daemon-reload && systemctl restart docker.service`


## 验证docker push 到Harbor ok吗？
k8s的各个节点用docker命令能否push镜像到harbor
然后在harbor网页上看镜像是否已经传上来了。
具体参考，前面的文章。

## 测试k8s与harbor连接
k8s从harbor得到镜像并运行pod（减轻了外网压力、方便了共享。）

```
kubectl --help
kubectl run nginx-deployment --image=harbor.ccbjb.com.cn/library/myapp:v1 -port=80 --replicas=1 
kubectl get deployment
kubectl get rs
kubectl get pod
kubectl get pod -o wide
curl 上一个命令的私有IP
curl 私有IP/hostname.html //获取pod容器的hostname

到运行节点
docker ps -a 
```

到harbor上，看该镜像的下载次数+=1

## 方便容灾和扩容

### 容灾测试
```bash
kubectl delete pod 刚刚的get pod获得的pod id
kubectl get pod //会发现又生成了一个pod，并且id不同
```

### 扩容测试
```bash
kubectl get deployment //获得deployment name
kubectl scale --replicas=3 deployment/deployment的name
kubectl get pod //能看到3个pod
kubectl get pod -o wide //能看到3个pod运行在不同的节点上
```

### 再验证容灾
```
kubectl get pod
kubectl delete pod "刚刚的get pod获得的pod id"
kubectl get pod //还是三个pod，会发现又生成了一个pod, id不同
```

### 怎么访问这三台pod？

#### 只能内部能访问的方式：类型ClusterIP
```bash
kubectl expose --help

// 访问服务的80端口，实际上访问呢的就是容器的8000
kubectl expose deployment nginx-deployment --port=30000 --target-port=80 //deployment名字自己查查

kubectl get svc

curl 【该service对应的clusterIP】:【30000】  // OK
//每次curl都是从不同的pod上传回来的。轮询三个不同的pod。负载均衡。

// 这个地址是内部地址，外部访问不了
http://【该service对应的clusterIP】:【30000】  // NG

kubectl get svc 
```
![](/images/2020-07-17-15-21-11.png)

#### 修改成让外部也能访问：类型NodePort

修改svc的type：ClusterIp→NodePort
```bash
kubectl edit svc nginx-deployment

修改：
Type：ClusterIp 为 Type：NodePort 

kubectl get svc
```

这样，**所有节点**都暴漏了该端口。

![](/images/2020-07-17-15-23-25.png)

```bash

netstat -anpt| grep 31859
//有了
```
外部访问：
http://192.168.3.105:31859 OK了
http://192.168.3.106:31859 OK了
http://192.168.3.107:31859 OK了
