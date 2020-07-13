---
layout: post
title: 【GitLab】Docker和docker-compose部署GitLab（第一天）
---

一个是docker部署gitlab，另一个是docker-compose部署gitlab
我尝试的是docker-compose。安装到192.168.3.111上。

## docker 部署gitlab
这个视频仅仅10分钟，非常非常简洁。
就是按照官方文档docker方式安装gitlab。
volumen目录定义一下就完事了。
https://www.bilibili.com/video/BV1Eb411N7sJ?from=search&seid=7154314386199342873


---

## docker compose 部署gitlab
https://www.bilibili.com/video/BV1NJ41137nE?from=search&seid=18101917166552153507

### 用compose搭建gitlab单点服务器（成功了）
参考：gitlab官网
[install gitlab by docker，docker-compose，swarm](https://docs.gitlab.com/omnibus/docker/)
[k8s安装部署gitlab](https://docs.gitlab.com/ee/install/README.html#installing-gitlab-on-kubernetes-via-the-gitlab-helm-charts)
docker-compose方法
    Install GitLab using Docker Compose
    With Docker Compose you can easily configure, install, and upgrade your Docker-based GitLab installation:

   1. Install Docker Compose.
   2. Create a docker-compose.yml file (or download an example):
    ```
    web:
    image: 'gitlab/gitlab-ce:latest'
    restart: always
    hostname: '192.168.3.111' 
    environment:
        GITLAB_OMNIBUS_CONFIG: |
        external_url 'https://192.168.3.111'
        # Add any other gitlab.rb configuration here, each on its own line
    ports:
        - '80:80'
        - '443:443'
        - '2222:22' # 注意，一般centos7启动的时候22端口就是被占用的。所以改成2222.
    volumes:
        - '$GITLAB_HOME/config:/etc/gitlab'
        - '$GITLAB_HOME/logs:/var/log/gitlab'
        - '$GITLAB_HOME/data:/var/opt/gitlab'
    ```
   3. Make sure you are in the same directory as docker-compose.yml and start GitLab:

    `docker-compose up -d`
   4. 稍等2分钟
      验证gitlab启动：https://192.168.3.111 
      

### 配置邮件服务器
##### 【配置公司自己的SMTP服务】
```
[root@centos111 config]# docker ps
CONTAINER ID        IMAGE                     COMMAND             CREATED             STATUS
          PORTS                                                            NAMES
191b986289c6        gitlab/gitlab-ce:latest   "/assets/wrapper"   About an hour ago   Up About an hour (healthy)   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:2222->22/tcp   root_web_1

[root@centos111 config]# pwd
/srv/gitlab/config
[root@centos111 config]# ls
gitlab.rb            ssh_host_ecdsa_key      ssh_host_ed25519_key.pub  ssl
gitlab.rb.bak        ssh_host_ecdsa_key.pub  ssh_host_rsa_key          trusted-certs
gitlab-secrets.json  ssh_host_ed25519_key    ssh_host_rsa_key.pub
```
该目录是gitlab的数据卷，等同于进入gitlab容器修改/etc/gitlab/gitlab.rb

`vi gitlab.rd `
```
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "服务器IP"
gitlab_rails['smtp_port'] = 25
gitlab_rails['smtp_user_name'] = "shirx@ccbjb.com.cn"
gitlab_rails['smtp_password'] = "密码"
gitlab_rails['smtp_domain'] = "ccbjb.com.cn"
gitlab_rails['smtp_authentication'] = "login"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['smtp_tls'] = false
gitlab_rails['gitlab_email_from'] = 'shirx@ccbjb.com.cn'
gitlab_rails['gitlab_email_reply_to'] = 'shirx@ccbjb.com.cn'
```
进入gitlab容器，让配置生效 

```
docker exec -it root_web_1 /bin/bash

gitlab-ctl reconfigure

```

报错，见TroubleShooting的Bug1.解决，再次运行。OK
进入控制台

```
gitlab-rails console  

//执行发送邮件测试命令
Notify.test_email("个人邮箱@qq.com","title","text").deliver_now
```

结果：
```

=> #<Mail::Message:70335281250660, Multipart: false, Headers: <Date: Thu, 02 Jul 2020 06:33:34 +0000>, <From: GitLab <shirx@ccbjb.com.cn>>, <Reply-To: GitLab <shirx@ccbjb.com.cn>>, <To: shirx@ccbjb.com.cn>, <Message-ID: <5efd7fbee008b_1f8d3ff84ced39a4792f2@gitlab.mail>>, <Subject: title>, <Mime-Version: 1.0>, <Content-Type: text/html; charset=UTF-8>, <Content-Transfer-Encoding: 7bit>, <Auto-Submitted: auto-generated>, <X-Auto-Response-Suppress: All>>

```

邮件也在手机上真实收到了。

其它SMTP配置，[参考官方](https://docs.gitlab.com/omnibus/settings/smtp.html)：

### TroubleShooting:
#####[Bug1]
```
There was an error running gitlab-ctl reconfigure:

letsencrypt_certificate[gitlab.ccbjb.com.cn] (letsencrypt::http_authorization line 5) had an error: RuntimeError: acme_certificate[staging] (/opt/gitlab/embedded/cookbooks/cache/cookbooks/letsencrypt/resources/certificate.rb line 25) had an error: RuntimeError: ruby_block[create certificate for gitlab.ccbjb.com.cn] (/opt/gitlab/embedded/cookbooks/cache/cookbooks/acme/resources/certificate.rb line 108) had an error: RuntimeError: [gitlab.ccbjb.com.cn] Validation failed, unable to request certificate
```
##### [解决]
原因：gitlab的bug（[bug 5000 ](https://gitlab.com/gitlab-org/omnibus-gitlab/-/issues/5000)）
为了使Gitlab正常工作，/etc/gitlab/gitlab.rb必须将其更改为letsencrypt['enable'] = false并重新运行 gitlab-ctl reconfigure. 

```
=> #<Mail::Message:70335281250660, Multipart: false, Headers: <Date: Thu, 02 Jul 2020 06:33:34 +0000>, <From: GitLab <shirx@ccbjb.com.cn>>, <Reply-To: GitLab <shirx@ccbjb.com.cn>>, <To: shirx@ccbjb.com.cn>, <Message-ID: <5efd7fbee008b_1f8d3ff84ced39a4792f2@gitlab.mail>>, <Subject: title>, <Mime-Version: 1.0>, <Content-Type: text/html; charset=UTF-8>, <Content-Transfer-Encoding: 7bit>,
 <Auto-Submitted: auto-generated>, <X-Auto-Response-Suppress: All>>
```