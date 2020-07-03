
# centos7 安装docker-compose 

官方文档：
https://docs.docker.com/compose/install/

一共就这么三步。大概三分钟
`sudo curl -L "https://github.com/docker/compose/releases/download/1.26.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose`

```zsh
[root@centos108 ~]# sudo curl -L "https://github.com/docker/compose/releases/download/1.26.1/docker-comp ose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   638  100   638    0     0    301      0  0:00:02  0:00:02 --:--:--   301
100 11.6M  100 11.6M    0     0   112k      0  0:01:46  0:01:46 --:--:--  142k


[root@centos108 ~]# chmod +x /usr/local/bin/docker-compose

[root@centos108 ~]# ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

[root@centos108 ~]# docker-compose --version
docker-compose version 1.26.1, build f216ddbf
```
