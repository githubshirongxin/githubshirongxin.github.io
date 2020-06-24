---
layout: post
---

# user vscode edit Github blog（write Github blog with vscode markdown plugins (Paste Image,Markdown All in one , markdown preview enhanced)）

## 1,Create a Github blog
1. search “Jekyll Now” repository in Github.
2. fork to your repository
3. edit _config.yml
   + name 随意
   + description 随意
4. add .md file to _posts
   + filename must YYYY-MM-DD-title.md
5. access your  github blog
   + Repositories → 选择你的repository → setting
   + rename 必须是你的github的用户名，否则你的重新编译。
      + （查看github用户名方法：右上角图标→Signed in as就是）
   + setting页向下拉“GitHub Pages”有blog链接



## 2, 编写md
6. 编写好的md放到_post/目录，可以创建子目录，只要文件名字系的符合要求就行
7. md所用到的文件放到/images/下
8. 本地vscode编写md，里面的图像文件需要注意一下。

**vscode 写md**
### git clone到本地
进入到项目根目录，右键vscode打开
目录结构：
 ![](/images/2020-06-24-14-03-23.png)
### vscode 安装插件
  + paste image
  + markdown all in one
  + markdown preview enhanced

### 配置paste image插件
让截图生成到/images/下面
md里图片的代码`![](/images/2020-06-24-14-04-01.png)`
这样就能很方便的用git直接传到github，而不用修改md的图片路径了。
![](/images/2020-06-24-14-04-01.png)
![](/images/2020-06-24-14-04-13.png)
![](/images/2020-06-24-14-04-44.png)
![](/images/2020-06-24-14-04-59.png)
![](/images/2020-06-24-14-05-22.png)

### 编写md
略

### 确认preview
点击下图的 ![](/images/2020-06-24-14-07-47.png)图标
![](/images/2020-06-24-14-07-29.png)

vscode的右半部会显示md的预览


### 确认github blog
`https://githubshirongxin.github.io/`
每张图都显示了。

### 最后，GIT上传所有图像和md文件