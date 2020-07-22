---
layout: post
title: 【vuepress】vuepress （四）github关联
---


## 4. 和github关联

指南→部署→[Github](https://vuepress.vuejs.org/zh/guide/deploy.html#github-pages)


### 新建 project/deploy.sh
```
#!/usr/bin/env sh

# 确保脚本抛出遇到的错误
set -e

# 生成静态文件
npm run build

# 进入生成的文件夹
cd docs/.vuepress/dist
git init
git add -A
git commit -m 'deploy'

git push -f git@github.com:githubshirongxin/vuepress.git master:gh-pages
cd -

# 如果发布到 https://<USERNAME>.github.io
# git push -u origin master

# 如果发布到 https://<USERNAME>.github.io/<REPO>
# git push -f git@github.com:<USERNAME>/<REPO>.git master:gh-pages
```

#### 问题：这样dist/都发布到vuepress库，但是其他文件没有放到github管理。

- 所有内容如docs库
在/project下`git init`创建·git目录。然后设置远程orgin
` git remote add origin git@github.com:githubshirongxin/docs.git master`
这样在project/下执行`git push` 就会把所有内容都push到docs库。

- dist内容如vuepress库
而在/project/docs/.vuepress/dist/下 `git init` 创建了origin
` git remote add origin git@github.com:githubshirongxin/vuepress.git master:b-branche`
在该目录下执行`git push`就只会把/dist下的内容发布到vuepress库中。

- 这两个目录的.git/互相并不影响，非常独立。使用的时候注意在哪个目录执行，执行的位置很关键。

### 修改package.json
```
"scripts": {
    "dev": "vuepress dev docs",
    "build": "vuepress build docs",
    "deploy": "bash deploy.sh"
  },
```

### 修改config.js的base
base：库名
这样访问的时候https://githubshirongxin.github.io/库名/
```
module.exports = {
    base: "/vuepress",
```

https://githubshirongxin.github.io/vuepress/

### 运行deploy
`npm run deploy`

查看github的vuepress库，都是dist/的内容
![](/images/2020-07-22-18-43-43.png)


细看<head><meta>都是/vuepress/assets/css...
![](/images/2020-07-22-18-44-18.png)
