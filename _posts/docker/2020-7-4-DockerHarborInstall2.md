---
layout: post
title: 【docker】nginx反向代理harbor两节点共享NFS存储、DB，弱可用Harobor部署（后续改善）
---
共享存储，NFS以及DB仍旧有单点故障。Nginx也有单点故障。以后：DB可以做成集群。nginx可以做成主从。NFS换成ceph。

感谢原作者akiya的分享。这篇文章写得已经算是不错了。至少给了一个尝试的方案。我略加整理，并解决了一下小Bug。亲测可以push。
https://juejin.im/post/5d973e246fb9a04dfa0963fb


下面是思路：
nginx负责负载均衡，nginx提供https。
nginx后面的两个harobr使用http服务。
网页访问的时候访问最前端的nginx。
客户端需要docker的daemon.json中加入最前端nginx的domain。

![](/images/2020-07-16-10-50-44.png)

如果最终生产环境集群中服务器较多，依赖做完LB的Harbor也无法完全达到需求时，可以使用如下架构，部署下级Harbor节点从主节点同步镜像，然后再分发给生产服务器。
![](/images/2020-07-16-10-51-12.png)

## 环境
IP|用途|安装的服务
|---|---|---|
192.168.3.120|NFS服务器，Nginx服务器，Harbor存储层|NFS、Redis、PostgressSQL、Nginx、Docker、docker-compose
192.168.3.108|Harbor无状态节点|docker、docker-compose、harbor（http方式）
192.168.3.109|Harbor无状态节点|docker、docker-compose、harbor（http方式）

|软件|	版本|
|---|---|
Docker|	19.06.3-ce
docker-compose|	1.25.0-rc2
Harbor|	1.10.0
Nginx|	1.16.1
PostgreSQL|	9.6.14
Redis|	4.0.14

版本的确定，首先你下载一个Harbor，然后用下面方法确认好redis，postgress的版本。


## 1. 192.168.3.120 部署
上安装registry、clair、notarysigner、notaryserver、Redis、NFS、证书

### 怎么查看harbor用的redis版本
`docker exec -it redis /bin/bash`
```
redis [ ~ ]$ redis-server --version
Redis server v=4.0.14 sha=00000000:0 malloc=jemalloc-4.0.3 bits=64 build=1e2a11bfe971a20a
```

### 怎么查看harbor用的postgress版本
`docker exec -it harbor-db /bin/bash`
```
postgres [ / ]$ psql
psql (9.6.14)
Type "help" for help.

postgres=# select version();
 PostgreSQL 9.6.14 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 6.3.0, 64-bit
(1 row)
```

### 1.1 签发私有证书
#### 生成私钥
#### 正式生产环境建议使用商业证书！

##### 使用openssl工具生成一个RSA私钥
`# openssl genrsa -des3 -out harbor.key 2048`
```
Generating RSA private key, 2048 bit long modulus
.......................+++
......+++
e is 65537 (0x10001)
Enter pass phrase for harbor.key:                   # 输入一个至少4位的密码
Verifying - Enter pass phrase for harbor.key:       # 重复输入密码
复制代码删除harbor.key中的密码
# openssl rsa -in harbor.key -out harbor.key
Enter pass phrase for harbor.key:                 # 输入刚才创建时的密码
writing RSA key
```



##### 生成CSR（证书签名请求）
`# openssl req -new -key harbor.key -out harbor.csr`

```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:cn             # 国家
State or Province Name (full name) []:Sichuan    # 地区 
Locality Name (eg, city) [Default City]:Chengdu  # 城市
Organization Name (eg, company) [Default Company Ltd]:akiya  # 组织
Organizational Unit Name (eg, section) []:akiya  # 组织单位
Common Name (eg, your name or your server's hostname) []:akiya    # 常用名可填自己名字或域名
Email Address []:a@b.com                         # 邮件地址

Please enter the following 'extra' attributes
to be sent with your certificate request      
A challenge password []:       # 可留空
An optional company name []:   # 可留空
```


##### 生成自签名证书
注意：在使用自签名的临时证书时，浏览器会提示证书的颁发机构是未知的。
`# echo subjectAltName = IP:192.168.3.120 > extfile.cnf`
`# openssl x509 -req -days 365 -in harbor.csr -signkey harbor.key -out harbor.crt -extfile extfile.cnf`

```
Signature ok
subject=/C=cn/ST=Sichuan/L=Chengdu/O=akiya/OU=akiya/CN=akiya/emailAddress=a@b.com
Getting Private key
```

##### 存放证书
复制证书到/www/certs待用
`# mkdir -p /www/certs && cp harbor.crt harbor.key /www/certs`

## 1.2 Docker
##### 官方一键脚本安装

`# curl -sSL https://get.docker.com/ | sh`


##### 先安装必要的依赖环境
`# yum -y install yum-utils device-mapper-persistent-data lvm2`

##### 添加软件源信息使用阿里云源安装
`# yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo`

##### 更新缓存
`# yum makecache fast`

##### 安装Docker或安装指定版本Docker
```
# yum -y install docker
# yum -y install docker-ce-18.06.3.ce-3.el7
```

##### 查看Docker版本
```
# docker --version
Docker version 18.06.3-ce, build d7080c1
```

##### 修改Docker仓库为国内镜像站
```
# curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io
```

##### 启动Docker服务并添加至开机自启
```
# systemctl start docker
# systemctl enable docker
```


## 1.3 Compose
compose是Docker提供的一个命令行工具，用来定义和运行由多个容器组成的应用。使用compose，我们可以通过YAML文件声明式的定义应用程序的各个服务，并由单个命令完成应用的创建和启动。
由于国内政策原因，可能在海外网站上下载文件速度较慢，建议下载本地后上传至服务器
##### 下载docker-compose并赋予可执行权限
```
# curl -L https://github.com/docker/compose/releases/download/1.25.0-rc2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

# chmod +x /usr/local/bin/docker-compose

//查看compose本地版本
# docker-compose -v

docker-compose version 1.25.0-rc2, build 661ac20e
```



## 1.4 四个服务的安装
由于Harbor v1.9.0使用的是PostgreSQL，我们也同样独立部署一套PostgreSQL与Redis，此次演示使用Docker部署，实际生产环境按需要选择是否部署至宿主机。

##### PostgreSQL
通过查看已安装的Harbor v1.9.0单机版中运行的harbor-db容器可得知此次运行的PostgreSQL版本为9.6.14
```
# docker exec -it harbor-db /bin/bash
postgres [ / ]$ psql
psql (9.6.14)
Type "help" for help.

postgres=# select version();
                                    version
-------------------------------------------------------------------------------
 PostgreSQL 9.6.14 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 6.3.0, 64-bit
(1 row)
```

##### Redis
根据官方文档描述，使用Redis需要4.0.10-1.ph2版本。同样，此次我们为了演示也使用docker-compose来部署。
##### yaml脚本
那么我们使用同一版本的PostgreSQL与Redis，编写docker-compose.yml 内容如下：
```
# author:akiya
version: "3"

networks:
  harbor:
    driver: bridge

services:
  registry:
    image: postgres:9.6.14
    container_name: harbor-registry
    restart: always
    environment:
      POSTGRES_DB: registry
      POSTGRES_PASSWORD: root123
    volumes:
      - $PWD/postgres/registry:/var/lib/postgresql/data
    networks:
      - harbor
    ports:
      - 20010:5432
  clair:
    image: postgres:9.6.14
    container_name: harbor-clair
    restart: always
    environment:
      POSTGRES_DB: clair
      POSTGRES_PASSWORD: root123
    volumes:
      - $PWD/postgres/clair:/var/lib/postgresql/data
    networks:
      - harbor
    ports:
      - 20011:5432
  notarysigner:
    image: postgres:9.6.14
    container_name: harbor-notarysigner
    restart: always
    environment:
      POSTGRES_DB: notarysigner
      POSTGRES_PASSWORD: root123
    volumes:
      - $PWD/postgres/notarysigner:/var/lib/postgresql/data
    networks:
      - harbor
    ports:
      - 20012:5432
  notaryserver:
    image: postgres:9.6.14
    container_name: harbor-notaryserver
    restart: always
    environment:
      POSTGRES_DB: notaryserver
      POSTGRES_PASSWORD: root123
    volumes:
      - $PWD/postgres/notaryserver:/var/lib/postgresql/data
    networks:
      - harbor
    ports:
      - 20013:5432
  Redis:
    image: redis:4.0.10
    container_name: harbor-redis
    command: redis-server /usr/local/etc/redis/redis.conf
    restart: always
    volumes:
      - $PWD/redis/data:/data
      - $PWD/redis/redis.conf:/usr/local/etc/redis/redis.conf
    networks:
      - harbor
    ports:
      - 20000:6379
```
保存后我们用`docker-compose up -d`命令启动相应的容器.
并在防火墙中开放对应的端口
```
# docker-compose up -d
# firewall-cmd --zone=public --permanent --add-port=20000/tcp
# firewall-cmd --zone=public --permanent --add-port=20010/tcp
# firewall-cmd --zone=public --permanent --add-port=20011/tcp
# firewall-cmd --zone=public --permanent --add-port=20012/tcp
# firewall-cmd --zone=public --permanent --add-port=20013/tcp
# firewall-cmd --reload
```
> 我一般推荐关闭防火墙。在内网里面开启防火墙没有任何意义。

## 1.5 NFS
### 1.5.1 服务端
##### 创建NFS共享文件路径

`# mkdir -p /data/nfs`

##### 安装NFS（在安装完nfs-utils后，rpcbind默认是启动了的）
`# yum -y install nfs-utils rpcbind`

##### 启动NFS相关服务并设置开机启动

```
# systemctl start rpcbind
# systemctl enable rpcbind
# systemctl start nfs-server
# systemctl enable nfs-server
# systemctl start nfs-lock
# systemctl enable nfs-lock
# systemctl start nfs-idmap
# systemctl enable nfs-idmap
```

##### 使用如下命令像/etc/exports中添加配置
```
# echo '/data/nfs  10.0.0.0/8(rw,sync,no_root_squash)' >> /etc/exports
# exportfs -a      # 使exports的修改生效
```

##### 检查NFS共享目录是否正确
```
# showmount -e localhost
Export list for localhost:
/data/nfs 10.0.0.0/8
```

##### 放行防火墙相应服务
```
# firewall-cmd --add-service=nfs --permanent --zone=public
# firewall-cmd --add-service=mountd --permanent --zone=public
# firewall-cmd --add-service=rpc-bind --permanent --zone=public
# firewall-cmd --reload
```

### 1.5.2 客户端
##### 创建NFS挂载文件路径
`# mkdir /data`
##### 安装NFS
`# yum -y install nfs-utils`
##### 检查NFS远程共享目录是否存在
```
# showmount -e 192.168.3.120
Export list for 192.168.3.120:
/data/nfs 10.0.0.0/8
```
##### 挂载远程NFS共享文件路径
`# mount -t nfs 192.168.3.120:/data/nfs /data`

##### 添加到系统开机自动挂载
`# echo '192.168.3.120:/data/nfs /data nfs defaults 0 0' >> /etc/fstab`

##### 测试
```
在客户端上挂载完NFS后创建一个测试文件
# touch /data/test
然后切换到服务器查看是否存在
# ls /data/nfs/
test
```

## 1.6 Nginx
### 安装
##### 添加Nginx源
```
# rpm -ivh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
```

##### yum安装Nginx
`# yum -y install nginx`
##### 启动Nginx
```
# systemctl start nginx
# systemctl enable nginx
```

##### 编写配置
创建`/etc/nginx/conf.d/harbor.conf`文件，并写入如下内容

```json
upstream harbor {
    ip_hash;
    server 192.168.3.108:80;
    server 192.168.3.109:80;
}
server {
   listen       80;
   # 提供访问的域名或者IP
   server_name  harbor.ccbjb.com.cn;
   return      308 https://$host$request_uri;
}
server {
    listen  443 ssl;
    server_name harbor.ccbjb.com.cn;
    
    # SSL 证书
    ssl_certificate ./certs/harbor.crt;
    # SSL 私钥
    ssl_certificate_key ./certs/harbor.key;
    client_max_body_size 0;
    chunked_transfer_encoding on;

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_redirect off;
        proxy_ssl_verify off;
        proxy_ssl_session_reuse on;
        proxy_pass http://harbor;
        proxy_http_version 1.1;
    }
    location /v2/ {
        proxy_pass http://harbor/v2/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_ssl_verify off;
        proxy_ssl_session_reuse on;
        proxy_buffering off;
        proxy_request_buffering off;
    }
}
```
#### 启动Nginx
##### 验证Nginx配置正确性
`# nginx -t`
##### 平滑重启Nginx
`# nginx -s reload`
##### 开放防火墙80/443端口
```
# firewall-cmd --zone=public --permanent --add-port=80/tcp
# firewall-cmd --zone=public --permanent --add-port=443/tcp
# firewall-cmd --reload
```
### 问题处理
Nginx
##### 问题：使用自签证书时报错
```
emerg] 31815#31815: cannot load certificate "/www/certs/harbor.crt": BIO_new_file() failed (SSL: error:0200100D:system library:fopen:Permission denied:fopen('/www/certs/harbor.crt','r') error:2006D002:BIO routines:BIO_new_file:system lib)
```
##### 解决方法：创建/etc/nginx/certs路径，并复制证书到此路径
```
mkdir -p /etc/nginx/certs && cp /www/certs/ ./certs
// 修改harbor.conf中证书相关路径
# 刚才我们自己签发的证书
ssl_certificate ./certs/harbor.crt;
# 证书对应的私钥
ssl_certificate_key ./certs/harbor.key;
```


## 2. 两台Harbor节点安装Harbor
### 2.1 修改配置文件
#### 修改harbor.yml配置文件
`# vim harbor.yml`

主要配置参数如下，由于我们这里使用外置PostgreSQL与Redis所以直接注释掉database相关配置改用external_database与external_redis


```yml
# 修改为当前服务器内网IP地址即可
hostname: 192.168.3.108 # 另一台修改为192.168.3.109
# HTTP相关配置
http:
  port: 80
# HTTPS相关配置，这里由于我们会在前端加一个Nginx
# 所以我们直接使用HTTP，而在Nginx上做SSL
#https:
#  # HTTPS端口
#  port: 443
#  # TLS证书
#  certificate: /www/certs/harbor.crt
#  # TLS私钥
#  private_key: /www/certs/harbor.key
# 默认管理员密码
harbor_admin_password: Harbor12345
# Harbor DB配置，由于使用外部数据库，所以这里我们注释掉
# database:
#   password: root123
#   max_idle_conns: 50
#   max_open_conns: 100
...
# 外部PostgreSQL，由于Harbor使用了4个数据库，这里我们也需要对相应数据库地址进行配置
external_database:
  harbor:
    host: 192.168.3.120
    port: 20010
    db_name: registry
    username: postgres
    password: root123
    ssl_mode: disable
    max_idle_conns: 2
    max_open_conns: 0
  clair:
    host: 192.168.3.120
    port: 20011
    db_name: clair
    username: postgres
    password: root123
    ssl_mode: disable
  notary_signer:
    host: 192.168.3.120
    port: 20012
    db_name: notarysigner
    username: postgres
    password: root123
    ssl_mode: disable
  notary_server:
    host: 192.168.3.120
    port: 20013
    db_name: notaryserver
    username: postgres
    password: root123
    ssl_mode: disable
# 使用外部Redis，取消相应注释即可
external_redis:
  host: 192.168.3.120
  port: 20000
  password:
  registry_db_index: 1
  jobservice_db_index: 2
  chartmuseum_db_index: 3
```

### 2.2 安装Harbor

##### 生成harbor运行的必要文件（环境）以及docker-compose.yml文件；
执行后会通过网络获取Docker Image，建议提前修改好国内镜像站加速。
`# ./prepare`


`# ./install.sh`

##### 开放Harbor端口
```
# firewall-cmd --zone=public --permanent --add-port=80/tcp
# firewall-cmd --reload
```
> 推荐关闭防火墙，就不需要这步了


#### 以上就是原文的全部了。这里有一处漏记。
如果不处理，会有push镜像失败的问题。

### 2.3 修改harbor的nginx配置
###### 参考：
https://goharbor.io/docs/2.0.0/install-config/troubleshoot-installation/

> 使用nginx或负载平衡
如果Harbor在nginx代理或弹性负载平衡之后运行，请打开文件common/config/nginx/nginx.conf并搜索以下行。
proxy_set_header X-Forwarded-Proto $scheme;
如果代理已经有类似的设置，从部分删除
location /，
location /v2/
location /service/
和重新部署港湾。有关如何重新部署Harbor的说明，请参阅 重新配置Harbor和管理Harbor生命周期。

##### 首先，修改两个nginx节点的nginx配置文件
harbor安装目录（harbor.xml同级目录）`/root/harbor`，修改nginx的配置文件
`/root/harbor/common/config/nginx/nginx.conf` 三处，注意只改三处。
否则，push的时候会一直retring！！！！！！！！
<font color=red>该配置文件的注释会有误导，导致你都注释掉，就怎么也push不成功,  注意只改三处!</font>


```
 location / {
      proxy_pass http://portal/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

      # When setting up Harbor behind other proxy, such as an Nginx instance, remove the below line if the proxy already has similar settings.
      # proxy_set_header X-Forwarded-Proto $scheme;

      proxy_buffering off;
      proxy_request_buffering off;
    }
location /v2/ {
      proxy_pass http://core/v2/;
      proxy_set_header Host $http_host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

      # When setting up Harbor behind other proxy, such as an Nginx instance, remove the below line if the proxy already has similar settings.
      # proxy_set_header X-Forwarded-Proto $scheme;
      proxy_buffering off;
      proxy_request_buffering off;

      proxy_send_timeout 900;
      proxy_read_timeout 900;
    }
 location /service/ {
      proxy_pass http://core/service/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

      # When setting up Harbor behind other proxy, such as an Nginx instance, remove the below line if the proxy already has similar settings.
      # proxy_set_header X-Forwarded-Proto $scheme;

      proxy_buffering off;
      proxy_request_buffering off;
    }
```

##### 然后，重启harbor服务
```
docker-compose down
docker-compose up -d
```

## 3. dockers客户端测试push

##### 192.168.3.107 上测试push

###### 客户端上修改docker配置文件
```
# vi /etc/docker/daemon.json
{
 "insecure-registries": ["harbor.ccbjb.com.cn"]
}

// 修改/etc/docker/daemon.json完成后reload配置文件
# sudo systemctl daemon-reload

// 重启docker服务
# sudo systemctl restart docker.service

```
不修改login不进去，push不让

```
# docker login -u admin harbor.ccbjb.com.cn
password:
输入harbor.xml里Harbor12345（当然我已经改成别的了，输入你自己的密码）
Login Succeeded

# docker tag node harbor.ccbjb.com.cn/library/vue/js

# docker push harbor.ccbjb.com.cn/library/vue/js
The push refers to a repository [harbor.ccbjb.com.cn/library/vuejs/ci]
3cb56e88501e: Pushed
5fe5f08a709e: Pushed
99ad6967cf69: Pushed
2aa1c54ebc66: Pushed
89cb25530034: Pushed
c4b55423085b: Pushed
b5e9932b9936: Pushed
d5644f4d8741: Pushed
4a03ae8d3bee: Pushed
a9286fedbd63: Pushed
d50e7be1e737: Pushed
6b114a2dd6de: Pushed
bb9315db9240: Pushed
latest: digest: sha256:3b766dda613fcd2dce50e4e2ba6ef9ae0e322a52ca1fff6d9e466a06e2a8a0e6 size: 3063
```
##### 同一个网段192.168.3.*随便哪台机器
https://harbor.ccbjb.com.cn/
admin
Harbor12345
都能够看到已经上传的镜像文件。






---
# 以下：一次失败的把 Harbor安装在samba挂载上的经历（不要尝试）
修改docker卷位置
首先把公司分给我的raid盘挂载过来。
然后把harbor默认的dockers卷修改在这块磁盘上。这样至少存储上提高了点安全性。

## 1，作为samba的客户端挂载服务端的磁盘
[samba实现分布式磁盘](https://githubshirongxin.github.io/Smba%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E7%A3%81%E7%9B%98/)

已经挂完 /data_cjb/

- 确认一下挂载情况：
```
mount | grep data_cjb
或者
df -hT | grep data_cjb
或者
cat /etc/fstab
```

## 2，修改harbor/docker-compose.yml , harbor.yml
### harbor.yml
```
 # Log configurations
   location: /var/log/harbor # 不改了，这块丢就丢了。
 # The default data volume
 #  data_volume: /data #这得改成mount好的raid盘目录
 data_volume: /data_cjb/harbordata
```

### docker-compose.yml

所有/data/ 都改成/data_cjb/harbordata/

## 3，执行`./install.sh`

最后/root/harbor/prepare 执行的时候报错。
说容器内没有/data/secret/cert/目录没有permission denied

## 结论：samba共享目录上无法安装harbor
另外，我在nfs上共享目录上就能安装harbor。

---
