---
layout: post
title: vuepress gitlab internet
---

从gitlab的sample page中找到vuepress 
https://gitlab.com/pages/vuepress

然后fork到自己的账户下

## clone到本地改改

改改之后，默认是配置了外网公共的Runner进行编译出静态内容。

## 改改.gitlab-ci.yml
https://gitlab.com/help/user/project/pages/getting_started/pages_from_scratch.md

```
image: node:9.11.1

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

### Q: 为什么yarn run build 会生成一个public目录呢？在哪设置的？



## 写文章
在docs/下写文章md文件

git push到gitlab.com外部服务器

自动触发Runner对静态文本编译成浏览器可以识别的html，js,css.

发布到gitlab.com什么地方，然后随着gitlab服务器一起启动了起来


## blog的分类
base = 库名
username.gitlab.io/reponame // 这种是project级别的，所以被称作project blog
base = /
username.gitlab.io  //这种是账号级别的，所以被称作account blog
orgnazition.gitlab.io // 这种是组织级别的，所以被称作公司 blog。



## 改gitlab page的域名
在gitlab page里修改域名，account blog的形式。


git push，试试。