---
layout:post
title:gitlab ci
---

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

gitlab的CI流程
https://gitlab.ccbjb.com.cn/help/ci/README.md

[How GitLab CI/CD works.](https://gitlab.ccbjb.com.cn/help/ci/introduction/index.md#how-gitlab-cicd-works)

[Fundamental pipeline architectures.](https://gitlab.ccbjb.com.cn/help/ci/pipelines/pipeline_architectures.md)
+ basic pipline
+ Directed Acyclic Graph
+ Child/Parent Piplines

[GitLab CI/CD basic workflow.](https://gitlab.ccbjb.com.cn/help/ci/introduction/index.md#basic-cicd-workflow)

![](/images/2020-07-06-16-48-41.png)
![](/images/2020-07-06-16-50-10.png)

[Step-by-step guide for writing .gitlab-ci.yml for the first time.](https://gitlab.ccbjb.com.cn/help/user/project/pages/getting_started_part_four.md)

以上，没什么用

---
https://www.bilibili.com/video/BV1B7411P7jX?from=search&seid=574991024952038149

学习笔记
服务器1：gitlab
服务器2：docker gitlab-runner
项目代码：golang
Dockerfile
.gitlab-ci.yml

2.1为项目准备Runner机器
2.2 将Runner机器与giltlab从cicd`注册`，完成连接和大同

docker run -d --name gitlab-runner --restart always \
   -v /srv/gitlab-runner/config:/etc/gitlab-runner \
   -v /var/run/docker.sock:/var/run/docker.sock \
   gitlab/gitlab-runner:latest


https://blog.csdn.net/qingchuwudi/article/details/103089075