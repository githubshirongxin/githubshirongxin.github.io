---
layout: post
title: 【gitlab】gitlab runner 能够成功编译vuepress项目了，但是gitlab pages不会开启
---

## 修改gitlab配置，本地gitlab部署gitlab pages服务
3.111 上因为我是docker安装gitlab
```
< pages_external_url "https://gitlab.ccbjb.com.cn"
< gitlab_pages['enable'] = true
< gitlab_pages['inplace_chroot'] = true
```
我为了配置起作用
把docker容器删掉，然后重新启动服务
```
docker ps 
docker stop c16e87838ef2 && docker rm c16e87838ef2
docker-compose up -d
//查看启动情况
docker ps 
docker logs -f c16e87838ef2
```

## gitlab runner 安装配置

3.112作为gitlab runner的机器。使用docker方式来安装gitlab runner。


### 3.112上新建/root/enviroment/Dockerfile
```
FROM gitlab/gitlab-runner:latest
MAINTAINER shirx <shirx@ccbjb.com.cn>

# 安装yarn （可选，只有执行器为shell时才有点用）
RUN apt-get update
RUN apt install -y curl
RUN apt-get update && apt-get install -y gnupg2
RUN curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
RUN echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.
RUN apt-get update
RUN apt install -y yarn
```

- 不安装也无所谓，因为大多数都是用docker当作执行器，最常用
- 本地环境yarn，npm等等只有shell执行器的时候才有用，不方便

### /root下创建docker-compose.yml
```
[root@centos112 ~]# cat docker-compose.yml
version: '3.1'
services:
  gitlab-runner:
    build: enviroment # 调用该目录下的Dockerfile文件
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

这里的runners是跑了一次生成的，不创建这个文件，就跑步过去。

### 启动gitlab-runner
```
docker-compose up -d
docker-compose up --build -d
// 查看一下启动日志
docker ps
docker logs -f 容器ID或Name
```


## 尝试1：新建一个项目plain html项目
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


## 尝试2：原来的vuepress项目

### 写.gitlab-ci.yml
```
stages:
  - build

pages build job:
  stage: build
  cache:
    paths:
    - node_modules/

  script:
   - npm ci
   - npm run build

  artifacts:
    paths:
    - docs/.vuepress/dist

  only:
  - master
  ```

  参考：https://docs.gitlab.com/ee/user/project/pages/getting_started/pages_from_scratch.html
  


### 注册Runner与Gitlag工程联系上
`docker exec -it gitlab_runner gitlab-runner register`
输入gitlab项目中CICD里的Runner的url，token
Tags直接回车
执行器这里可以选shell。但是最好选docker，然后指定docker镜像
一般：
  - vuepress等需要node：latest
  - php
  - ruby
  - java
  - 等


`docker exec -it gitlab_runner gitlab-runner start`
有时候网页上Runner为灰色未连接状态，才需要执行一下这个。


### 修改vuepress md文件，并push
能看到 pipline编译md文件为html文件

#### github
这里最后更新到F:\temp\docs\.vuepress\dist
不过这是github的pages的目录。
gitlab不认，需要更新到与.gitlab-ci.yml同级的public/下才行

#### gitlab
gitlab的时候需要修改
- .gitlab-ci.yml把job名修改成"pages:"（其他job名网页上一概无法生成blog链接）
这样写的一定要用Docker 执行器来注册。
```
image: node:latest

pages:
  cache:
    paths:
    - node_modules/

  script:
   - yarn install
   - yarn run build 
 
  artifacts:
    paths:
    - public
  only:
  - master
```
- docs/.vuepress/config.js，需要修改dest: "public" ,必须为public，否则gitlab不认

```
const headConf = require("./config/headConf.js");
const pluginsConf = require("./config/pluginsConf");
const nav = require("./nav");
module.exports = {
  base: "/docs/",
  title: 'blog',
  dest: "public",
  description: '思想persistent',
  head: headConf,
  markdown: {
    lineNumbers: false // 代码块显示行号
  },
  plugins: pluginsConf,
  themeConfig: {
    lastUpdated: '更新时间',
    logo: '/logo.png',
    nav: nav,
    searchMaxSuggestions: 10
    /* 方案1：侧边栏只显示三组中的一组链接 */
    // sidebar: sidebarConf, //使用了vuepress plugin auto sidebar就不用这些了
  }
}

```
---

# 其他参考内容（可以不看）：

## rpm版 gitlab install
1.添加gitlab镜像
访问
https://packages.gitlab.com/gitlab/gitlab-ce/packages/ol/7/gitlab-ce-13.1.5-ce.0.el7.x86_64.rpm

```
curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo yum install gitlab-ce-13.1.5-ce.0.el7.x86_64

vim  /etc/gitlab/gitlab.rb
gitlab-ctl reconfigure
gitlab-ctl restart
```

```
查看与rpm包相关的文件和其他信息   rpm -qa | grep 包名
查询包是否被安装，命令：rpm -q 包名
删除软件包，命令：rpm -e 包名
```

## docker版gitlab runner install
其实完全可以这么从容地安装Runner。写docker-compose主要是为了更新的时候方便点。
```
docker run -d --name gitlab_runner --restart always \
     -v /srv/gitlab-runner/config:/etc/gitlab-runner \
     -v /var/run/docker.sock:/var/run/docker.sock \
     gitlab/gitlab-runner:latest

docker exec -it gitlab_runner gitlab-runner register
docker exec -it gitlab_runner gitlab-runner start
docker restart gitlab_runner
docker exec -it gitlab_runner gitlab-runner unregister 

## 进入容器看看
docker exec -it gitlab_runner /bin/sh
cd /home/gitlab-runner/builds/N1Bf5i3f/0/shirongxin
```
# 删掉所有容器，删掉指定镜像
```
docker stop $(docker ps -q) && docker rm $(docker ps -aq) 
docker rmi -f root_gitlab-runner
```
