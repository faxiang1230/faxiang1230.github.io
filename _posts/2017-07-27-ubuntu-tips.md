---
layout:     post
title:      "ubuntu使用小case"
date:       2017-07-26 19:50:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - ubuntu
---
# ubuntu使用
## 快速更换更新源
![image](../images/ubuntu-apt-source.png)
sudo apt-get update
## 获取软件源码
```
lh@server2:~$ which pure-ftpd
/usr/sbin/pure-ftpd
lh@server2:~$ dpkg -S /usr/sbin/pure-ftpd
pure-ftpd: /usr/sbin/pure-ftpd
lh@server2:~$apt-get source  pure-ftpd
```
