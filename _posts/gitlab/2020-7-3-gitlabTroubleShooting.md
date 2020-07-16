---
layout: post
title: 【GitLab】gitlab ci trouble shootting（持续更新中）
---

gitlab来做CICD时，出现的问题。在这里做个整理。
方便日后查找。

## 【问题】【已解决】：我的gitlab runner问题，成功注册，但是就一直pending
#### 【现象】：
gitlab-runner
> WARNING: Checking for jobs... failed                runner=gYEgXsvL status=couldn't execute POST against https://gitlab.ccbjb.com.cn/api/v4/jobs/request: Post https://gitlab.ccbjb.com.cn/api/v4/jobs/request: read tcp 172.20.0.3:42366->192.168.3.111:443: read: connection reset by peer


```
[root@centos111 ~]# docker exec -it root_gitlab-runner_1 /bin/bash

bash-5.0# gitlab-runner status
Runtime platform                                    arch=amd64 os=linux pid=28 revision=6fbc7474 version=13.1.1
gitlab-runner: Service is not running.
```

```
bash-5.0# gitlab-runner run
Runtime platform                                    arch=amd64 os=linux pid=33 revision=6fbc7474 version=13.1.1
Starting multi-runner from /etc/gitlab-runner/config.toml...  builds=0
Running in system-mode.

Configuration loaded                                builds=0
listen_address not defined, metrics & debug endpoints disabled  builds=0
[session_server].listen_address not defined, session endpoints disabled  builds=0
```

#### 【原因】 问题在与.git-ci.yml中tags没有指定！**
所以runner成功注册但状态一直是“pending”，然后超时，failed
见：https://blog.csdn.net/qingchuwudi/article/details/103089075
#### 【解决】：** .gitlab-ci.yml中增加tags：与runner的tags对应上。



## 【问题】【没办法解决】 docker-compose方式安装gitlab，安装在sambada挂载盘上面
/srv是192.168.3.122的挂载盘，挂载192.168.100.11/shi目录。
`docker-compose up -d`

```bash
alpine: Pulling from gitlab/gitlab-runner
21c83c524219: Pull complete
4de95b93bf1d: Pull complete
651e812d526b: Pull complete
1e95a28388dc: Pull complete
eb618f17f262: Pull complete
e3d8175fc390: Pull complete
3fbd3623c394: Pull complete
f88b016862ab: Pull complete
Digest: sha256:b46c3c1f8a46e14cc54c480684f67e7b73e94f21e1d8eb9fd9d2a436b405ccbf
Status: Downloaded newer image for gitlab/gitlab-runner:alpine
Creating gitlab ... error

ERROR: for gitlab  Cannot start service gitlab: error while creating mount source path '/srv/gitlab/logs': chown /srv/gitlab/logs: permission denied

ERROR: for gitlab  Cannot start service gitlab: error while creating mount source path '/srv/gitlab/logs': chown /srv/gitlab/logs: permission denied
ERROR: Encountered errors while bringing up the project.

#[root@centos122 ~]# ll /srv
总用量 0
drwxrwxrwx. 1 1032 users 0 7月  14 13:59 data
drwxrwxrwx. 1 1032 users 0 7月  16 15:00 gitlab
```  

#### 【无法解决】：sambda挂载盘就是不支持gitlab安装。




  