---
layout: post
title: 【GitLab】Docker和docker-compose部署GitLab 单机版（第二天补记，成功经验）
---
单机版。写个shell每天备份该机器/srv目录到公司的raid盘。先公司内部凑合用。
存储高可用的，我现在还不知道怎么做。

完全基于[[Docker折腾记: (3)Docker Compose构建Gitlab,从配置(https,邮箱验证)到基本可用]](https://juejin.im/post/5b55bf1c6fb9a04fac0d13b3)

成功部署在192.168.3.111上，验证好用。


# gitlab docker-compose install


这才是我想要的gitlab docker-compose部署的方式
配置写得非常好

## 版本二的docker-compose.xml，基本可用
唯独一点：2222:22 → 22:22 .否则`git  clone git@gitlab.ccbjb.com.cn:shirongxin/edusite.git` 永远提示输入密码，输入什么也不对。具体做法下面有。

### 修改docker-compose.yml 
```
version: '3.6'
services:
  gitlab:
    container_name: gitlab
    privileged: true
    image: gitlab/gitlab-ce:latest
    restart: always
    hostname: 'gitlab.ccbjb.com.cn'
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://gitlab.ccbjb.com.cn'
        gitlab_rails['smtp_enable'] = true
        gitlab_rails['smtp_address'] = "（邮件服务器IP）.100.5"
        gitlab_rails['smtp_port'] = 25
        gitlab_rails['smtp_user_name'] = "shirx@ccbjb.com.cn"
        gitlab_rails['smtp_password'] = "密码"
        gitlab_rails['smtp_domain'] = "ccbjb.com.cn"
        gitlab_rails['smtp_authentication'] = "login"
        gitlab_rails['smtp_enable_starttls_auto'] = true
        gitlab_rails['smtp_tls'] = false
        gitlab_rails['gitlab_email_from'] = 'shirx@ccbjb.com.cn'
        gitlab_rails['gitlab_email_reply_to'] = 'shirx@ccbjb.com.cn'
        nginx['enable'] = true
        nginx['client_max_body_size'] = '250m'
        nginx['redirect_http_to_https'] = true
        nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.ccbjb.com.cn_chain.crt"
        nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.ccbjb.com.cn_key.key"
        nginx['ssl_ciphers'] = "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256"
        nginx['ssl_prefer_server_ciphers'] = "on"
        nginx['ssl_protocols'] = "TLSv1.1 TLSv1.2"
        nginx['ssl_session_cache'] = "builtin:1000  shared:SSL:10m"
        nginx['listen_addresses'] = ["0.0.0.0"]
        nginx['http2_enabled'] = true
    ports:
      - '80:80'
      - '443:443'
      - '22:22'
    volumes:
      - '/srv/gitlab/config:/etc/gitlab'
      - '/srv/gitlab/logs:/var/log/gitlab'
      - '/srv/gitlab/data:/var/opt/gitlab'
  gitlab-runner:
    image: gitlab/gitlab-runner:alpine
```
### 1. 重新安装的时候，因为gitlab的数据已经有一些更新了。并备份了一下数据。
`tar cvf srv.tar /srv`

如果需要在别的机器安装，或者本3.111机器因为某种原因重新安装虚拟机了，
需要把srv.tar备份到某处，千万不要放到一台机器上，太危险。
另外，也需要至少每天备份一次。

先这么手动备份把，之后我准备把gitlab安装成主从备份模式。
成功之后我准备安装到k8s上，多节点，数据也是多节点。这样就不怕单点失败了。


### 2. 运行docker-compose之前，免费证书申请
https://freessl.org/apply?domains=gitlab.ccbjb.com.cn&product=buypass01&from=

 ![](/images/2020-07-03-16-27-18.png)
 ![](/images/2020-07-03-16-27-33.png)
 ![](/images/2020-07-03-16-28-18.png)
+ KeyManage 选择导出的时候选择**ngix方式**，因为gitlab的证书是配置在nginx里的。
+ 并用DNS验证的方式验证。下载下来压缩包解压后是两个文件，
gitlab.ccbjb.com.cn_chain.crt
gitlab.ccbjb.com.cn_key.key
+ 拷贝到/etc/gitlab/ssl/(容器内视角)
或拷贝到宿主机srv/gitlab/config/ssl（宿主机视角）

+ 修改docker-compose.yml对应的证书的部分。




### 3. 把宿主机3.111的ssh端口22空出来。供容器映射用，必须这么做。很重要！！！！
参考： [Centos7如何修改ssh默认端口](https://blog.csdn.net/ZanShichun/article/details/78029561?utm_source=blogxgwz5)
#### 3.1 停掉firewall，停掉selinux  

```
systemctl stop firewalld
systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
systemctl stop NetworkManager
systemctl disable NetworkManager
```

#### 3.2 增加2020为ssh端口
```
$ vi /etc/ssh/sshd_config
 Port 22  
 Port 2020

//重启sshd服务
systemctl restart sshd.service

//验证2020为ssh
$ ssh -p 2020 root@192.168.3.111 //成功

//删除Port 22 这一行，只留下我们的Port 2020
vim /etc/ssh/sshd_config

//然后重启ssh服务
systemctl restart sshd.service
```

####  3.3 必须重新启动docker服务。否则执行docker-compose会报如下错误：(因为停掉firewall) 

```
Error response from daemon: driver failed programming external connectivity
 on endpoint jenkins (a8ea15bf9b3dbed599d059d638f79f9dd5e875556c39bfb41e6563d3feedb81b):
  (iptables failed: iptables --wait -t nat -A DOCKER -p tcp -d 0/0 --dport 50000 -j DNAT
 --to-destination 172.18.0.6:50000 ! -i br-031aa3930383: iptables: No chain/target/match
 by that name.
```
光看这个报错: iptables: No chain/target/match by that name，就能够看出是跟iptables有关,如果再启动docker service的时候网关是关闭的，那么docker管理网络的时候就不会操作网管的配置（chain docker），然后网关重新启动了，导致docker network无法对新container进行网络配置，也就是没有网管的操作权限，做重启处理.

#### 3.4【处理】: 必须重启docker服务，这步必须做！！
```
service docker restart
或
systemctl restart  docker
```

## 4.然后，启动docker-compose
`docker-compose up -d ` 
即可。

## 5. 验证gitlab运行状态
#### 5.1 docker logs 看容器启动日志
 ![](/images/2020-07-03-20-08-21.png)
 `docker logs -f 32de52f7ae22` 查看容器日志，看到gitlab启动情况

现在能看到,别的都正常！

【残留课题】：
```
ERROR: Failed to load config stat /etc/gitlab-runner/config.toml: no such file or directory  builds=0
```

#### 5.2 画面正常启动，原来的内容都保留着。
 ![](/images/2020-07-03-20-20-38.png)

#### 5.3 git clone ssh看看是否正常
```
srx@DESKTOP-UQ03MQF MINGW64 /c/work/gitlab.ccbjb.com.cn
$ git clone git@gitlab.ccbjb.com.cn:shirongxin/edusite.git
```
>Cloning into 'edusite'...
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ECDSA key sent by the remote host is
SHA256:wsq50fWrV3xiqsKGem5DUWqGCglzR/WP1F3FhfBVH2A.
Please contact your system administrator.
Add correct host key in /c/Users/srx/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /c/Users/srx/.ssh/known_hosts:12
ECDSA host key for gitlab.ccbjb.com.cn has changed and you have requested strict checking.
Host key verification failed.
fatal: Could not read from remote repository.
Please make sure you have the correct access rights
and the repository exists.

处理：把自己用户c:/users/srx/.ssh/know_hosts里删掉
gitlab.ccbjb.com.cn这一行

```
srx@DESKTOP-UQ03MQF MINGW64 /c/work/gitlab.ccbjb.com.cn
$ git clone git@gitlab.ccbjb.com.cn:shirongxin/edusite.git
```

> Cloning into 'edusite'...
The authenticity of host 'gitlab.ccbjb.com.cn (192.168.3.111)' can't be established.
ECDSA key fingerprint is SHA256:wsq50fWrV3xiqsKGem5DUWqGCglzR/WP1F3FhfBVH2A.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'gitlab.ccbjb.com.cn' (ECDSA) to the list of known hosts.
Warning: the ECDSA host key for 'gitlab.ccbjb.com.cn' differs from the key for the IP address '192.168.3.111'
Offending key for IP in /c/Users/srx/.ssh/known_hosts:11
Are you sure you want to continue connecting (yes/no)? yes
remote: Enumerating objects: 38, done.
remote: Counting objects: 100% (38/38), done.
remote: Compressing objects: 100% (28/28), done.
remote: Total 38 (delta 4), reused 38 (delta 4), pack-reused 0
Receiving objects: 100% (38/38), 4.67 KiB | 239.00 KiB/s, done.
Resolving deltas: 100% (4/4), done.

ssh方式clone OK了

能够免密了。（公钥已经放到了gitlab的SSH-KEY）中了。

#### 5.4 git clone https 验证
```
$ git clone https://gitlab.ccbjb.com.cn/shirongxin/edusite.git
```
>Cloning into 'edusite'...
remote: Enumerating objects: 38, done.
remote: Counting objects: 100% (38/38), done.
remote: Compressing objects: 100% (28/28), done.
remote: Total 38 (delta 4), reused 38 (delta 4), pack-reused 0
Unpacking objects: 100% (38/38), done.


成功！
至此，就差防止gitlab单点故障问题了。

### 6.TODO：
 + 方案1： 主从gitlab实时备份
 + 方案2： k8s运行gitlab多节点共享数据，共享数据多点同步