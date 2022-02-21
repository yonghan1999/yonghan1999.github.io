---
layout: post
title: Mac(M1)安装JDK8启动慢解决方案
date: 2021-04-02
categories: 技术
tags: macOS
---

**系统版本**

- macOS Big Sur 11.2.2
- Zulu JDK 8

##### 什么问题

同样的网络环境，同一个SpringBoot项目

- 在win10下启动，需要10秒左右
- 在ubuntu18.04下启动，需要7秒左右
- 在macOS下启动，竟然需要23多秒

**解决方案**

- 打开终端输入`hostname`，查看你的mac的主机名称，把它给复制下来

- 修改`/etc/hosts`文件，大家应该都知道这个文件是做什么的

- 没修改前应该是这个样子

  ```python
  ##
  # Host Database
  #
  # localhost is used to configure the loopback interface
  # when the system is booting.  Do not change this entry.
  ##
  127.0.0.1	localhost
  255.255.255.255	broadcasthost
  ::1	localhost
  ```

- 把它改成这个样子

  ```python
  ##
  # Host Database
  #
  # localhost is used to configure the loopback interface
  # when the system is booting.  Do not change this entry.
  ##
  127.0.0.1	localhost lyxdeMacBook-Pro.local
  255.255.255.255	broadcasthost
  ::1	localhost HandeMacBook-Air.local # 'HandeMacBook-Air.local'  替换为你的主机名
  ```

- `lyxdeMacBook-Pro.local`即是前边复制下的主机名

- 好了，再运行项目就可以了，2到4秒，终于舒服了

只有在 JDK8 才会出现该问题，在 JDK11 正常。