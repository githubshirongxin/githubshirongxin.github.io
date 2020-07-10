---
layout: post
title: docker本地库 + harbor
---

+ 【Q13】docker本地仓库与harbor。本地仓库的意义，harbor的意义？
+ 【A】本地仓库意义：
     1. 其它机器不开外网也能下载镜像。
     2. 局域网传输速度远远大于外网传输。快。
     3. 公司的研发成功，没必要传到github上。方便大家共享一些半成品镜像。
     4. 这些镜像只对公司内部人员有意义。外人也不愿意看。
---
+ 【Q14】如果大家都有网的话，大家又不介意快慢，本地镜像库有还有什么意义？
     1. 镜像文件的共享。     
---
+ 【Q15】 使用了本地docker仓库，对大家有什么影响？
     1. 本地镜像库就是个register镜像启动的容器。ip:port
     2. 其它人使用上，就是pull的时候镜像名字前面多了个 ip:port/xxx:banben
     3. 其他人使用上，也就是commit的时候，镜像名字前多个 ip:port/
     4. 用不用docker login？
     5. 使用docker search的时候用不用加ip：port？
     6. docker tag的时候多了个ip：port
     7. 默认创建在/tmp/registry下。
---
+ 【Q16】使用官方DockerHub，与使用本地仓库有什么不同？
     1. docker login
     2. docker tag 本地库/镜像tag  dockerhubId/镜像tag
     3. docker push dockerhubId/镜像tag 
<br>
---
+ 【Q17】本地镜像和harbor什么关系？
     1. 本地仓库是docker官方的registry镜像
     2. harbor是vmware公司的基于registry的管理UI
     3. 提供了额外的功能  
<br>    
---
+ 【Q18】harbor的部署
     https://www.cnblogs.com/kevingrace/p/6547616.html
     这里不建议使用kubernetes来部署, 原因是镜像仓库非常重要, 尽量保证部署和维护的简洁性, 因此这里直接使用compose的方式进行部署。官方提供3种部署Harbor的方式:
      1）在线安装: 从Docker Hub下载Harbor的镜像来安装, 由于Docker Hub比较慢, 建议Docker配置好加速器。
      2）离线安装: 这种方式应对与部署主机没联网的情况使用。需要提前下载离线安装包: harbor-offline-installer-.tgz 到本地
      3）OVA安装: 这个主要用vCentor环境是使用
      

<br>
---

+ 【Q19】本地库安装
     1. https://www.cnblogs.com/L-dongf/p/11028731.html
     2. 为docker配置加速器
      ```[root@harbor-node ~]# mkdir -p /etc/docker
      [root@harbor-node ~]# cat /etc/docker/daemon.json
      {
        "registry-mirrors": ["https://v5d7kh0f.mirror.aliyuncs.com"]
      }
      ```
     3. 下载最新的online install 包。
      https://github.com/goharbor/harbor/releases  
      `wget  https://github.com/goharbor/harbor/releases/download/v2.0.1/harbor-online-installer-v2.0.1.tgz`
     4. 升级docker版本。https://www.jianshu.com/p/6e5da590aeda
     5.  [harbor]# `vim harbor.yml` ,hostname, db_password,harbor_admin_password改成自己的。
     6. [harbor]# `./install.sh`
      > Error happened in config validation...
           ERROR:root:Error: The protocol is https but attribute ssl_cert is not set
     7. Harbor配置https认证 。https://www.cnblogs.com/Dev0ps/p/10566398.html
     8.   修改两处：harbor.yml
       ```
         # http related config
           http:
             # port for http, default is 80. If https enabled, this port will redirect to https port
             port: 80

           # https related config
           https:
             # https port for harbor, default is 443
             port: 443
             # The path of cert and key files for nginx
             certificate: /root/ca/192.168.3.108.crt
             private_key: /root/ca/192.168.3.108.key
      
     9. 再执行 ./install.sh
     10. ✔ ----Harbor has been installed and started successfully.----
     11. 测试：https://192.168.3.108 ,用户名admin和密码查看harbor.yml
     12.  ![](/images/2020-07-07-18-16-39.png)
   <font color=red>【残课题】：harbor配置邮件服务器，提示错误“验证邮件服务器失败，错误: failed to ping email server”</font>
---
+ 【Q20】如何使用harbor？
+  【A】
    +  harbor的主机名 192.168.3.108  
    +  `在harbor上创建项目edusite`。
    +  现在其它机器上 改变tag成harbor库的，`docker tag xxxx:banbenhao 192.168.3.108/edusite/xxxx:banbenhao`
    +  在其它机器上`docker push 192.168.3.108/edusite/xxxx:banbenhao  `
      ```
         [root@centos3 ~]# docker push 192.168.3.108/edusite/node
         The push refers to a repository [192.168.3.108/edusite/node]
         Get https://192.168.3.108/v1/_ping: x509: certificate signed by unknown authority
      ```

        +  解决：

      ```
      vim /etc/docker/daemon.json
      { 
        "insecure-registries": ["registry.svc.xxx.cn"]
      }
      #systemctl restart docker
      ```
      
      ```
      unauthorized: unauthorized to access repository: edusite/node, action: push: unauthorized to access repository: edusite/node, action: push
      ```
      + 解决： harbor上，进入该项目，为该项目添加成员，shirx，maoat，mengxt
        再push，成功！！！ 


    +  其它机器上pull ，`docker pull 192.168.3.108/edusite/xxx:banbenhao `
---
+ 【Q21】本地镜像服务器harbor如何配置成分布式？（防止单点故障） ※★★
     1. https://www.cnblogs.com/liangyuntao-ts/p/11199887.html
     2. 双主从模式（推荐，灾难恢复简单，扩展容易），其它cephfs和k8s模式灾难恢复太苦难。
     3. 
---
+ 【Q22】harbor库的主从节点，但是数据放到一个点，DNS服务器（NFS）的一个目录。主从同时挂载这个目录。我觉得不好，这样，NFS那台服务器如果硬盘坏掉，岂不变成无法恢复的单点故障了？
     1. https://blog.csdn.net/weixin_43304804/article/details/86507467
     2. 写得倒是挺详细的，可惜我觉得称不上“高可用” 