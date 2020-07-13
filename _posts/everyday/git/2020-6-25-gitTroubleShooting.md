---
layout: post
title: 【git】Git Trouble Shooting（持续更新）
---

记录git使用过程中遇到的问题。持续更新。

## 现象：git commit -m"hello" 仍旧提示“Aborting commit due to empty commit message.”
【解决】：.git/config 下user和email清空了。我在vi补上。

## 现象：The authenticity of host 'github.com (13.250.177.223)' can't be established.
```
$ git push
The authenticity of host 'github.com (13.250.177.223)' can't be established.
```
【解决】ping ping 13.250.177.223 ,已经ping不通。估计是被墙了。
https://site.ip138.com/raw.Githubusercontent.com/ 输入github.com
放到/etc/hosts里。然后执行
`git push`

```
Warning: Permanently added the RSA host key for IP address '192.30.255.113' to the list of known hosts.
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

【继续解决：】
```
$ ssh-keygen -t rsa -b 4096 -C "shirx@ccbjb.com.cn"
一路回车
Generating public/private rsa key pair.
Enter file in which to save the key (/c/Users/Administrator/.ssh/id_rsa):
/c/Users/Administrator/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /c/Users/Administrator/.ssh/id_rsa
Your public key has been saved in /c/Users/Administrator/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:OC1g39FYwEiTVqjIe2YAlrRjt86vwyzadIxgrkvA+Q8 shirx@ccbjb.com.cn
The key's randomart image is:

cat /c/Users/Administrator/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQCoB20IkrlOigkoh0iqVwE5xWFUCnzhVSsGYSI/YxJogEzqBguI5eCnMnOGbM0+tGtNIhexO308MOxjSz90/LEdw9EoZtfmq1mUyJZ80/略

拷贝到github的SSH key中

```
再试 OK了
```
$ git push
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Delta compression using up to 16 threads
Compressing objects: 100% (5/5), done.
Writing objects: 100% (5/5), 461 bytes | 461.00 KiB/s, done.
Total 5 (delta 2), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To github.com:githubshirongxin/githubshirongxin.github.io.git
   844ceab..933d3b1  master -> master
```