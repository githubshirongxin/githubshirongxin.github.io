---
layout: post
title: 【gitlab】gitlab runner 能够成功编译vuepress项目了，但是gitlab pages不会开启
---

## gitlab runner 安装配置

3.112作为gitlab runner的机器。使用docker方式来安装gitlab runner。


### 3.112上新建/root/enviroment/Dockerfile
```
FROM gitlab/gitlab-runner:latest
MAINTAINER shirx <shirx@ccbjb.com.cn>

# 安装yarn
RUN apt-get update
RUN apt install -y curl
RUN apt-get update && apt-get install -y gnupg2
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
RUN echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.
RUN apt-get update
RUN apt install -y yarn
```

### /root下创建docker-compose.yml
```
[root@centos112 ~]# cat docker-compose.yml
version: '3.1'
services:
  gitlab-runner:
    build: enviroment
    restart: always
    container_name: gitlab_runner
    privileged: true
    volumes:
      - /usr/local/docker/runner/config:/etc/gitlab-runner
      - /var/run/docker.sock:/var/run/docker.sock
```

### 准备好config.toml
```
[root@centos112 ~]# cat /usr/local/docker/runner/config/config.toml
concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "3.112 runner for docs"
  url = "https://gitlab.ccbjb.com.cn"
  token = "N1Bf5i3fkh_5nYTrVbDD"
  executor = "shell"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
[root@centos112 ~]#
```

### 启动gitlab-runner
```
docker-compose up -d
docker-compose up --build -d
```

## 新建一个项目
create from template，选Pages/plain text.
自动带.gitlib-ci.yml
自动带public/index.html


### register
```
docker exec -it gitlab_runner gitlab-runner register
docker exec -it gitlab_runner gitlab-runner start
```

然后去gitlab上去看。已经绿了。

### 修改public/index.html
pipeline 自动编译
pages：弹出链接

修改一下windows的system32/driver/etc/hosts
gitlab ip 弹出的那个新域名
`192.168.3.111 shirongxin.gitlab.ccbjb.com.cn`

## 访问pages链接
可以看到页面了。



