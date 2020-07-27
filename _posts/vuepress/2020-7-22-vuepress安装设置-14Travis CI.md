---
layout: post
title: 【vuepress】vuepress （十四）使用Travis CI
---

部署→Github→[TravisCI](https://www.vuepress.cn/guide/deploy.html#github-pages)

在项目的根目录创建一个名为 .travis.yml 的文件；
```
language: node_js
node_js:
  - lts/*
install:
  - npm ci # npm ci
script:
  - npm run build # npm run docs:build
deploy:
  provider: pages
  skip_cleanup: true
  local_dir: docs/.vuepress/dist
  github_token: $GITHUB_TOKEN # 在 GitHub 中生成，用于允许 Travis 向你的仓库推送代码。在 Travis 的项目设置页面进行配置，设置为 secure variable
  keep_history: true
  on:
    branch: master
```

https://travis-ci.com/
github用户登陆
会显示github所有工程，在docs后点击【Trigger abuild】
注意不是vuepress库，因为vuepress只有dist没有编译需要的源代码。


github生成GitHubTOKEN，一个用户只有一个。
![](/images/2020-07-23-11-04-50.png)
![](/images/2020-07-23-11-06-19.png)

把这个变量定义在travis-ci网站
![](/images/2020-07-23-11-06-51.png)

随便改改某个md

别忘了把.gitignore的私密信息注释，也push到github。
例如.gitignore
secret.js

否则Travis-ci编译不成功

### 修改某md后git push

### 去Travis-ci网页上查看

Current
https://travis-ci.com/github/githubshirongxin/docs
![](/images/2020-07-23-11-20-29.png)

### Github pages
![](/images/2020-07-23-11-21-08.png)
 Your site is published at https://githubshirongxin.github.io/docs/

网页也都反映了md的修改。


### 我还真是不明白
docs这样的目录统统传上去之后，travis-ci怎么编译的。
如果这样就可以的化，就不需要两个库了。
一个docs库：/project
一个vuepress库：/dist