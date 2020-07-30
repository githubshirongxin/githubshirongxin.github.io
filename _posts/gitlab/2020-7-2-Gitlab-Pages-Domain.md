---
layout: post
title:  【gitlab】gitlab pages 设置域名
---

参考两篇文章：
https://gitlab.com/gitlab-org/gitlab-foss/-/issues/53820
https://docs.gitlab.com/ee/administration/pages/index.html#custom-domains


### 思路：
公司内部想搭建个人博客和团队用博客、公司整体博客。
现在看，gitlab本地部署pages，最容易实现的就是个人博客。

#### http还是https
当然是https好了，但是https需要公钥和私钥。
- 自签名的公钥和私钥每次访问网页都会提示“不安全”，然后再点击链接强行访问。不够友好。
- 如果使用FreeSSL.org等

我就不信了，难道其他公司都使用通配符域名做的公钥私钥吗？


#### 个人博客
某人自建了vuepress工程，本地gitlab上发布，Runner自动编译之后，网页URL ：http://某人用户名.cjb.io/工程名。


##### 问题
- 域名申请，通配符域名申请，域名解析生效的过程。
- 数字证书颁发的过程。
- 两者有何顺序和关联？

这个视频完美地解决了我的问题。
https://www.bilibili.com/video/BV1BA411i7mF?from=search&seid=2045925171014425635

总结如下：
逻辑：
1. 域名能够申请成功，说明域名唯一性合法性（已付费）。
2. 域名能够添加到NameServer，说明域名已经申请成功。
3. TXT记录的DNS验证，说明域名申请成功即唯一性合法性。
4. 只要域名申请成功了，就可以颁发证书。证书里对于该网站的一切信息都是作者自己写的，CA不会去验证。
5. CA的证书仅仅证明该域名已经被验证了。是的，因为SSL证书有两种：域名型证书和企业型证书。
6. 域名型证书CA只负责认证这是个正常的域名。
7. 企业型证书CA除了负责认证这是个正常的域名之外，还要人工审核该企业的信息，确保域名就是该企业的之后，才给你颁发证书。
8. CA的对企业性证书更有意义。

![](/docs/images/2020-07-30-09-38-07.png)

##### HTTPS的数字证书验证原理

HTTPS的数字证书验证原理
https://blog.csdn.net/liuxingrong666/article/details/83869161


讲得最好的颁发证书的过程了。可以自己搭建CA。
https://blog.csdn.net/rookiewyh/article/details/102949016


SSL证书申请，分SSL证书的类型，
- 最基本的域名型SSL证书，只需要让CA验证这是个合法域名就行，方法是DNS里有TXT记录。CA会生成TXT记录让你写入DNS。让后在CA页面上点验证，返回了CA期待的值才给你下载证书。
- 企业性证书，CA除了验证域名合法性之外，还需要人工验证企业。
- 企业增强型证书，CA验证的就更多了。

![](/docs/images/2020-07-30-09-45-07.png)


##### 总结：
1. SSL证书颁发过程中，只有TXT记录的DNS验证与域名申请有点关系。
顺序是：
先域名申请
TXT记录 或 A记录记入DNS
也就是说，TXT记录的DNS验证，根本不需要A记录。申请个域名，不添加A记录，也能做DNS验证，也就是SSL证书申请颁发。
2. 如果使用本地LDNS的话，CA权威机构是没办法给你颁发SSL证书的，因为DNS解析验证过不去，根本找不到你在局域网里的LDNS。
   所以，想要CA权威的证书，就得用公网DNS！

##### 最后的方案：
不申请新的域名了。沿用公司在外网DNS注册的gitlab.ccbjb.com.cn,
在外网DNS上增加泛域名 *.gitlab.ccbjb.com.cn 解析到 192.168.3.111 IP上。
https方式也不用了，因为申请泛域名每年450元的价格倒是不贵，但考虑到仅仅是公司内部使用，不对外公开的gitlab page。用http也行。

所以，最后定*.gitlab.ccbjb.com.cn 外网DNS增加泛域名的A记录 + 通讯方式使用http。


##### 域名申请+域名解析的过程

这是两个过程 域名申请。就是占用了一个url
域名解析，就是在某个DNS上绑定url与IP。

一般申请域名和DNS网站：DNSPOD（中国），狗爹（外国）。

https://cloud.tencent.com/developer/article/1401677
总结：就是分两步，
注册一个域名。
第二步，绑定域名和ip，再DNS里增加A记录。
注册域名的服务器可以是DNS服务器。
也可以选择其他DNS服务器（NameServer）。


###### （题外话） TODO：
- 我发现运营和运维有一些专用工具。我需要专门学习学习。
- 宝塔面板，运营人员使用 https://www.bt.cn/



