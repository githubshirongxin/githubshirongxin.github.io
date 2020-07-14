---
layout: post
title: 【vscode】【remote-container插件】进入container像windows一样开发
---
vscode进入container，开发container里的文件。不用虚拟机，开发完就扔掉。程序可以持久化到本地。
不用安装hyperv，下一个docker，`docker pull centos`就拥有了70M大小的linux。

启动起来。
```
λ docker container run  --name=mycentos --volume=/data:/data/  --restart=always  -d centos
df29e4e8fb0212ec7e90ced90f01d297d50c95434e6a6b59338db0fab4534f16
```

然后ctl+shift+p：remote-containers：Attach to Running Container
![](/images/2020-07-14-11-18-17.png)

然后，选择一个container，进入。
就像在本地一样，进入container，开发container里的文件。

这些container是在vscode所在机器上的容器。

如果是远端运行的容器，就没办法了吗？

