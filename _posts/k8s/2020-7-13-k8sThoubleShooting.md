---
layout: post
title: 【k8s】k8s trouble shootting【持续更新中】
---

k8s运维过程中遇到的问题会持续更新。

## 问题一览表
### 1. [Q]: kubectl get nodes  ROLES <none>
### 解决: https://blog.csdn.net/textdemo123/article/details/104795784/
   给节点打上标签即可。`kubectl label node k8s-node1 node-role.kubernetes.io/worker=worker` k8s-node1为node的hostname


### 2. [Q] kubectl get nodes . The connection to the server 192.168.170.132:8080 was refused - did you specify the right host or port?

### 解决：
   https://blog.csdn.net/qq_24046745/article/details/94405188
   出现这个问题的原因是kubectl命令需要使用kubernetes-admin来运行，
   解决方法如下：
   
   将主节点中的【/etc/kubernetes/admin.conf】文件拷贝到从节点相同目录下，
   然后配置环境变量：
   **centos1** (master): 
      `scp /etc/kubernetes/admin.conf root@centos2:/etc/kubernetes/admin.conf`

   **centos2** (worker): 
      `echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bash_profile`
      立即生效
      `source ~/.bash_profile`
      在运行kubectl命令就成功了
   