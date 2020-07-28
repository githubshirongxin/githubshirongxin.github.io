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
  


### register
```
docker exec -it gitlab_runner gitlab-runner register
docker exec -it gitlab_runner gitlab-runner start
```

### 修改vuepress md文件，并push
能看到 pipline编译md文件为html文件

```
Running with gitlab-runner 13.2.1 (efa30e33)
   on 3.112 runner for docs N1Bf5i3f
Preparing the "shell" executor
00:00
 Using Shell executor...
Preparing environment
00:00
 Running on centos112...
Getting source from Git repository
00:01
 Fetching changes with git depth set to 50...
 重新初始化现存的 Git 版本库于 /home/gitlab-runner/builds/N1Bf5i3f/0/shirongxin/docs/.git/
 Checking out 47a14b65 as master...
 正删除 docs/.vuepress/dist/assets/css/0.styles.023d9b0a.css
 正删除 docs/.vuepress/dist/assets/js/10.81551f42.js
 正删除 docs/.vuepress/dist/assets/js/11.57d177ce.js
 正删除 docs/.vuepress/dist/assets/js/12.d71321c0.js
 正删除 docs/.vuepress/dist/assets/js/13.cca00ccc.js
 正删除 docs/.vuepress/dist/assets/js/14.f7f7ac64.js
 正删除 docs/.vuepress/dist/assets/js/15.92f2bc87.js
 正删除 docs/.vuepress/dist/assets/js/16.5641f890.js
 正删除 docs/.vuepress/dist/assets/js/17.664f02a3.js
 正删除 docs/.vuepress/dist/assets/js/18.a9f3acce.js
 正删除 docs/.vuepress/dist/assets/js/19.56426047.js
 正删除 docs/.vuepress/dist/assets/js/2.9b178d7b.js
 正删除 docs/.vuepress/dist/assets/js/20.83a4ce65.js
 正删除 docs/.vuepress/dist/assets/js/21.63248d14.js
 正删除 docs/.vuepress/dist/assets/js/22.4947826f.js
 正删除 docs/.vuepress/dist/assets/js/23.f1cce4c7.js
 正删除 docs/.vuepress/dist/assets/js/24.4a7086b3.js
 正删除 docs/.vuepress/dist/assets/js/25.0e247dc6.js
 正删除 docs/.vuepress/dist/assets/js/26.92d1f56a.js
 正删除 docs/.vuepress/dist/assets/js/27.93753bf2.js
 正删除 docs/.vuepress/dist/assets/js/28.4753ab37.js
 正删除 docs/.vuepress/dist/assets/js/29.0ae28ef7.js
 正删除 docs/.vuepress/dist/assets/js/3.ce270750.js
 正删除 docs/.vuepress/dist/assets/js/30.c3411470.js
 正删除 docs/.vuepress/dist/assets/js/31.d2eafd61.js
 正删除 docs/.vuepress/dist/assets/js/32.ad8b85c5.js
 正删除 docs/.vuepress/dist/assets/js/4.562b61e0.js
 正删除 docs/.vuepress/dist/assets/js/5.22cc210d.js
 正删除 docs/.vuepress/dist/assets/js/6.b04edcaa.js
 正删除 docs/.vuepress/dist/assets/js/7.0435da16.js
 正删除 docs/.vuepress/dist/assets/js/8.9495a4ca.js
 正删除 docs/.vuepress/dist/assets/js/9.9591550f.js
 正删除 docs/.vuepress/dist/assets/js/app.8ac3c304.js
 正删除 node_modules/
 Skipping Git submodules setup
Restoring cache
00:06
 Checking cache for default...
 Runtime platform                                    arch=amd64 os=linux pid=24137 revision=efa30e33 version=13.2.1
 No URL provided, cache will not be downloaded from shared cache server. Instead a local version of cache will be extracted. 
 Successfully extracted cache
Executing "step_script" stage of the job script
00:47
 $ npm ci
 npm WARN prepare removing existing node_modules/ before installation
 > fsevents@1.2.13 install /home/gitlab-runner/builds/N1Bf5i3f/0/shirongxin/docs/node_modules/fsevents
 > node install.js
 Skipping 'fsevents' build as platform linux is not supported
 > core-js@3.6.5 postinstall /home/gitlab-runner/builds/N1Bf5i3f/0/shirongxin/docs/node_modules/@vuepress/core/node_modules/core-js
 > node -e "try{require('./postinstall')}catch(e){}"
 > core-js@3.6.5 postinstall /home/gitlab-runner/builds/N1Bf5i3f/0/shirongxin/docs/node_modules/@vue/babel-preset-app/node_modules/core-js
 > node -e "try{require('./postinstall')}catch(e){}"
 > vuepress@1.5.2 postinstall /home/gitlab-runner/builds/N1Bf5i3f/0/shirongxin/docs/node_modules/vuepress
 > opencollective-postinstall || exit 0
 > core-js@2.6.11 postinstall /home/gitlab-runner/builds/N1Bf5i3f/0/shirongxin/docs/node_modules/core-js
 > node -e "try{require('./postinstall')}catch(e){}"
 added 1230 packages in 23.852s
 $ npm run build
 > project@1.0.0 build /home/gitlab-runner/builds/N1Bf5i3f/0/shirongxin/docs
 > vuepress build docs
 wait Extracting site metadata...
 tip Apply local theme at /home/gitlab-runner/builds/N1Bf5i3f/0/shirongxin/docs/docs/.vuepress/theme...
 tip Apply theme local (extends @vuepress/theme-default) ...
 tip Apply plugin container (i.e. "vuepress-plugin-container") ...
 tip Apply plugin @vuepress/last-updated (i.e. "@vuepress/plugin-last-updated") ...
 tip Apply plugin @vuepress/register-components (i.e. "@vuepress/plugin-register-components") ...
 tip Apply plugin @vuepress/active-header-links (i.e. "@vuepress/plugin-active-header-links") ...
 tip Apply plugin @vuepress/search (i.e. "@vuepress/plugin-search") ...
 tip Apply plugin @vuepress/nprogress (i.e. "@vuepress/plugin-nprogress") ...
 tip Apply plugin @vssue/vssue (i.e. "@vssue/vuepress-plugin-vssue") ...
 tip Apply plugin auto-sidebar (i.e. "vuepress-plugin-auto-sidebar") ...
 tip Apply plugin @vuepress/medium-zoom (i.e. "@vuepress/plugin-medium-zoom") ...
 [info] [webpackbar] Compiling Client
 [info] [webpackbar] Compiling Server
 [success] [webpackbar] Client: Compiled successfully in 18.67s
 [success] [webpackbar] Server: Compiled successfully in 18.68s
 wait Rendering static HTML...
 success Generated static files in docs/.vuepress/dist.
Saving cache
00:07
 Creating cache default...
 Runtime platform                                    arch=amd64 os=linux pid=24281 revision=efa30e33 version=13.2.1
 node_modules/: found 27156 matching files and directories 
 No URL provided, cache will be not uploaded to shared cache server. Cache will be stored only locally. 
 Created cache
Uploading artifacts for successful job
00:00
 Uploading artifacts...
 Runtime platform                                    arch=amd64 os=linux pid=24308 revision=efa30e33 version=13.2.1
 docs/.vuepress/dist: found 90 matching files and directories 
 Uploading artifacts as "archive" to coordinator... ok  id=235 responseStatus=201 Created token=GV6xLs3Y
 Job succeeded
 ```


最后更新到F:\temp\docs\.vuepress\dist
不过这是github的pages的目录。
gitlab不认，需要更新到与.gitlab-ci.yml同级的public/下才行


## 修改.gitlab-ci.yml
最后生成到public

