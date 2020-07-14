---
layout: post
title: 【vscode】【remote-ssh插件】用vscode代替命令行开发（写shell）
---

像windows一样用IDE开发linux文件。
linux上开发也可以用vscode的编辑器方便地编写远端文件。
这样不熟悉vim的人可以更方便了。
也有winscp能达到相同的功能。但总感觉low，实际上vscode的远端开发功能除了好看，本质上和winscp没什么不同。

步骤：

打开vscode界面

## 1，下载remote-ssh相关插件
![](/images/2020-07-14-09-11-19.png)

## 2，打开远端窗口：
点左下角图标，
![](/images/2020-07-14-09-11-43.png)
或者
ctl+shift+p，输入remote-ssh：
![](/images/2020-07-14-09-13-22.png)

### 选add new host
输入 `ssh root@192.168.3.108`
![](/images/2020-07-14-09-13-58.png)

### 再ctl+shift+p ，选connect host
![](/images/2020-07-14-09-15-19.png)

输入密码,打开了新的vscode工作界面。
![](/images/2020-07-14-09-16-19.png)

### 选择工作目录
点open folder，
![](/images/2020-07-14-09-16-44.png)
选择工作目录
![](/images/2020-07-14-09-17-12.png)

### 新的工作台可以用了。
这就可以工作了。像windows一样开发linux。
![](/images/2020-07-14-09-17-52.png)


