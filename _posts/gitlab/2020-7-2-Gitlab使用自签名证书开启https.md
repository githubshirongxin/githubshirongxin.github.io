---
layout: post
title: 【Gitlab】使用自签名证书开启Https
---

作者：HanlyJiang
链接：https://www.jianshu.com/p/4111534b339f



## 建立认证目录
```
sudo mkdir -p /etc/gitlab/ssl
sudo chmod 700 /etc/gitlab/ssl
```


## 建立证书
### 建立自签名证书
#### 1. 创建 Private Key
`sudo openssl genrsa -des3 -out /etc/gitlab/ssl/gitlab.domain.com.key 2048`

记住输入的密码（Pass phrase）

#### 2. 生成 Certificate Request
`sudo openssl req -new -key /etc/gitlab/ssl/gitlab.domain.com.key -out /etc/gitlab/ssl/gitlab.domain.com.csr`

```
Enter Country Name US
Enter State or Province Full Name
Enter City Name
Enter Organization Name
Enter Company Name
Enter Organizational Unit Name
Enter server hostname i.e. URL gitlab.domain.com
Enter Admin Email Address
Skip Challenge Password (Hit Enter)
Skip Optional Company Name (Hit Enter)
```

#### 注: 第一步和第二部可以合并成一条命令
`openssl req -nodes -newkey rsa:2048 -keyout gitlab.domain.com.key -out gitlab.domain.com.csr`

#### 3. 移除Private Key 中的密码短语
```
sudo cp -v /etc/gitlab/ssl/gitlab.domain.com.{key,original}
sudo openssl rsa -in /etc/gitlab/ssl/gitlab.domain.com.original -out /etc/gitlab/ssl/gitlab.domain.com.key
sudo rm -v /etc/gitlab/ssl/gitlab.domain.com.original
```

#### 4. 创建证书
```
sudo openssl x509 -req -days 1460 -in /etc/gitlab/ssl/gitlab.domain.com.csr -signkey /etc/gitlab/ssl/gitlab.domain.com.key -out /etc/gitlab/ssl/gitlab.domain.com.crt
```

#### 5. 移除证书请求文件
```
sudo rm -v /etc/gitlab/ssl/gitlab.domain.com.csr    
// 设置文件权限
sudo chmod 600 /etc/gitlab/ssl/gitlab.domain.com.*
```

#### gitlab 配置更改

`sudo vim /etc/gitlab/gitlab.rb`

```
external_url 'https://gitlab.domain.com'
可选配置：
gitlab 网站https：
nginx['redirect_http_to_https'] = true
gitlab ci 网站https：
ci_nginx['redirect_http_to_https'] = true

```
#### 复制证书到gitlab目录：
` sudo cp /etc/gitlab/ssl/etc/gitlab/ssl/gitlab.domain.com.crt /etc/gitlab/trusted-certs/`


#### gitlab重新配置+更新：
```
sudo gitlab-ctl reconfigure
sudo gitlab-ctl restart
```

#### Z. 注意事项
###### Z1. gitlab如何找到对应的证书？
/etc/gitlab/gitlab.rb 配置的external_url 如果是https://gitlab.domain.com 那么gitlab会自动去找/etc/gitlab/ssl/ 目录中的gitlab.domain.com.crt 和gitlab.domain.com.key 文件，所以如果需要更换external_url，可以采取如下办法：

将原先的crt和key文件改名为新的external_url 对应的域名
直接通过配置指定证书和key文件的路径
nginx['ssl_certificate'] = "/etc/gitlab/ssl/gitlab.domain.com.crt"
　　
nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/gitlab.domain.com.key"
别忘了reconfigure

