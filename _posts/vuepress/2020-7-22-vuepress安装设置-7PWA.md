---
layout: post
title: 【vuepress】vuepress （七）PWA离线功能
---

注意:需要在本地安装vuepress，否则PWA功能实现不了

## 7 PWA离线功能

vuepress指南→插件→[pws](https://www.vuepress.cn/plugin/official/plugin-pwa.html)
`npm install -D @vuepress/plugin-pwa`


config.js的分配置pluginConf.js

```javascript
 '@vuepress/pwa': {
      serviceWorker: true,
      updatePopup: {
        '/': {
          message: "New content is available.",
          buttonText: "Refresh"
        },
        '/zh/': {
          message: "发现新内容可用",
          buttonText: "刷新"
        }
    }
```

为了让你的网站完全地兼容 PWA，你需要:
在 .vuepress/public 提供 Manifest 和 icons
在 .vuepress/config.js 添加正確的 head links(参见下面例子).
更多细节，请参见 [MDN docs about the Web App Manifest](https://developer.mozilla.org/en-US/docs/Web/Manifest).

![](/images/2020-07-23-08-03-55.png)
### 从vuepress包里拿manifest.json放到pulic下过来改改。
- 改改name，shortname
- 改改图片的位置增加/vuepress/.(config.js里的base=vuepress对manifest不起作用，必须的自己改。)
  否则，浏览器有错误。
```
{
  "name": "cjb的资料库",
  "short_name": "cjb",
  "icons": [
    {
      "src": "/vuepress/icons/android-chrome-192x192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/vuepress/icons/android-chrome-512x512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ],
  "start_url": "/vuepress/index.html",
  "display": "standalone",
  "background_color": "#fff",
  "theme_color": "#3eaf7c"
}
```
### 从vuepress包里拿icons整个目录放到public下
![](/images/2020-07-23-08-16-57.png)


### headConf.js例子里拷贝过来就行。和上面的icons是能对应上
```
head: [
    ['link', { rel: 'icon', href: '/logo.png' }],
    ['link', { rel: 'manifest', href: '/manifest.json' }],
    ['meta', { name: 'theme-color', content: '#3eaf7c' }],
    ['meta', { name: 'apple-mobile-web-app-capable', content: 'yes' }],
    ['meta', { name: 'apple-mobile-web-app-status-bar-style', content: 'black' }],
    ['link', { rel: 'apple-touch-icon', href: '/icons/apple-touch-icon-152x152.png' }],
    ['link', { rel: 'mask-icon', href: '/icons/safari-pinned-tab.svg', color: '#3eaf7c' }],
    ['meta', { name: 'msapplication-TileImage', content: '/icons/msapplication-icon-144x144.png' }],
    ['meta', { name: 'msapplication-TileColor', content: '#000000' }]
  ],
```

## 编译
`npm run build`
```
tip Apply plugin @vuepress/pwa (i.e. "@vuepress/plugin-pwa") ...

√ Client
  Compiled successfully in 9.57s

√ Server
  Compiled successfully in 7.88s

wait Rendering static HTML...
wait Generating service worker...
success Generated static files in docs\.vuepress\dist.
```
### 编译将会生成dist/service-worker.js
![](/images/2020-07-23-08-28-13.png)

`npm run deploy`

## 都配置完成之后怎么测试呢？

必须在本地安装vuepress，否则无法生成servic-worker.js
![](/images/2020-07-23-08-47-35.png)
在浏览器F11，看到Cache里存储了这些。就OK了。说明断网也可以浏览这些存储过的东西了。

### 浏览器没error
之前浏览过的网页断网也能看

### 手机端
打开网页，提示你添加到主屏。然后浏览一通。断网。再浏览，发现都缓存下来了。不怕断网。仍旧能看一段时间！！
