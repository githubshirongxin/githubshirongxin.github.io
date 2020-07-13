---
layout: post
title: gitlab ci
---

## 我的gitlab runner问题，成功注册，但是就一直pending
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

---


**问题在与.git-ci.yml中tags没有指定！**
所以runner成功注册但状态一直是“pending”，然后超时，failed
见：https://blog.csdn.net/qingchuwudi/article/details/103089075
**解决：** .gitlab-ci.yml中增加tags：与runner的tags对应上。


  

  