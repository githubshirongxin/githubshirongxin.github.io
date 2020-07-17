---
layout: post
title: 【k8s】【视频教程70P】(三. K8s资源清单yaml文件讲解)
---
k8s资源，资源清单，常用字段说明，容器生命周期

## k8s 【资源】简介
资源实例化叫对象
集群资源分类（按生效范围）：

- 名称空间 < 集群级别
- 元数据型
两者不是一个标准。在两者之间。

### 名称空间级别：
只在次名称空间有效 kubeadm k8s 是kube-system
默认 -n default
在某个空间下看不到另一个空间的资源。

#### 1. 工作负载型资源
pod、ReplicaSet(通过标签管理pod创建)、Deployment、StatefulSet（有状态服务）、DaemonSet（运行pod主键）、Job（批处理）、CronJob

#### 2. 服务发现及负载均衡资源
SLB,Service,Ingress

#### 3. 配置与存储资源
Volume（存储卷）,CSI（容器存储接口，可以扩展第三方存储卷）

#### 4. 特殊类型的存储卷
ConfigMap（当配置中心用）,Secret(sfd),DownloadAPI(把外部环境的信息输出给容器)

###  集群级别：
在集群范围内有效 role
不管在哪个名字空间下，只要是这个集群，全集群可见。

NameSpace,Node,Role,ClusterRole,RoleBingding  ,ClusterRoleBinding

### 元数据级别：
通过指标进行操作 HPA（通过cpu使用率来扩展） 
PodTemplate,LimitRange

---

## 【资源清单】

yaml创建符合期望的pod，这个yaml叫资源清单

### yaml的格式：
- 不允许使用Tab，只允许使用空格
- 缩进的空格数不重要，只要相同层级的左侧对齐即可。
- #表示注释

### yaml支持的数据结构
- 对象
- 数组
- 纯量：单个不可再分的值

#### 对象：
```
name: shirx
age: 19
```

#### 数组：
```
aniaml
- Cat
- dog

animal: [cat,dog]
```

#### 复合类型:

```
language:
- Ruby
- perl
- python
websites:
Yaml: yaml.org
Ruby: ruby-lang.org
```

#### 纯量：
```
时间日期，数值
number: 12.30
isSet : true
parent: ~ # null
iso8601:2001-12-14t21:59
e: !!str 123 #强制转换成字符串
str: 这是字符串
str: '这是 字符串'
str: '这是 \n字符串' #不转义
str: "这是 \n字符串" #转义成换行
// 单引号中还有单引号,必须使用两个单引号转义
str: 'shirx''s dog'
// 第二行必须至少有单空格缩进.换行会被转为空格
str:字符串
  可以写成
  多行

// 也可以用|或> 保留换行
this: | 
is
a 
story

// |+ 表示保留换行符 |- 表示删除末尾换行符
str: this |-
```


---

## pod必须存在的属性
pod必须存在的属性
![](/images/2020-07-17-17-17-11.png)

有这些属性就能运行pod了。

## pod可选的属性
![](/images/2020-07-17-17-25-05.png)

imagePullPolicy : always 总是拉新镜像
command : 覆盖镜像里原来的指令
args: command的参数
workDir：跟docker的workdir一样

![](/images/2020-07-17-17-27-21.png)
ports
ports.name :端口名称
ports.hostPort : 主机上的端口
ports.protocol :

![](/images/2020-07-17-17-29-14.png)
![](/images/2020-07-17-17-30-25.png)
![](/images/2020-07-17-17-30-00.png)

用到的时候就明白了。

```
//讲上面的具体属性的。
kubectl explain --help
// 例如
kubectl explain pod.apiVersion
kubectl explain pod.spec
kubectl explain pod.metadata
```
```bash
vim pod.yaml

apiVersion: v1
kind: Pod # 必须大写P
metadata:
  name: myapp-pod
  namespace: default
  labels:
    app: myapp
    version: v1
spec:
  containers: 
  - name: app
    image: harbor.ccbjb.com.cn/library/myapp:v1
  - name: test
    image: harbor.ccbjb.com.cn/library/myapp:v1 # 可能端口被占用
```

`kubectl -f pod.yaml`

`kubectl get pod`

`kubectl describe pod mypodName`

`kubectl log myaap-pod -c test` # log后接pod名字，-c指定容器名字 ，看log

https://www.bilibili.com/video/BV1ME411g7EU?p=17 17:53 