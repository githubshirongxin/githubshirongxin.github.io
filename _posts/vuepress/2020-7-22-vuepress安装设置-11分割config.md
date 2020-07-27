---
layout: post
title: 【vuepress】vuepress （十一）分割config.js
---

为了好维护，把config.js拆分一下。仅此而已。


## 1，head配置文件
### 新建.vuepress/config目录
### 进入.vuepress/config目录
### 新建headConf.js
```
module.exports = [ // 注入到当前页面的 HTML <head> 中的标签 favicon.io
    ['link',{ rel: 'icon',href: '/favicon.ico'}],
    ['meta', { name: 'name', content: 'shirongxin' }],
    ['meta', { name: 'keywords', content: 'java,cobol,分布式存储' }], // 增加一个自定义的 favicon(网页标签的图标)

]
```
### 修改config.js
```
// 增加引入
const headConf = require("./config/headConf");

module.exports = {
...
// 修改head，使用引入的常量
head: headConf,
...

}

```

## 2，plugin配置文件
类似
![](/images/2020-07-22-20-32-44.png)
## 最后config.xml
```
const headConf = require("./config/headConf.js");
const pluginsConf = require("./config/pluginsConf");
const navConf = require("./config/navConf.js");
const sidebarConf = require("./config/sidebarConf.js");


module.exports = {
    base: "/vuepress",
    title: 'shirx blog',
    description: '我的个人网站',
    head: headConf,
    markdown: {
      lineNumbers: false // 代码块显示行号
    },
    plugins: pluginsConf,
    themeConfig: {
      lastUpdated: '更新时间',
      logo: '/logo.png',
      nav: navConf,
      /* 方案1：侧边栏只显示三组中的一组链接 */
       sidebar: sidebarConf,
    }
}
```

