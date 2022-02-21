---
layout: post
title: Mint Linux安装Docker踩坑指南
date: 2020-07-11
categories: 技术
tags: Linux
---

我家的服务器选用的Linux Mint系统，最近安装Docker的时候踩了一些小坑，但是总体还算顺利。
我们都知道Linux Mint系统是基于Ubuntu的，说实话用起来感觉还是很不错的，安装Docker到Ubuntu的方法**几乎**可以完全迁移到Mint上。
当然，问题就出在这个**几乎**上。

首先是正常安装各种依赖：

~~~shell
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
~~~

这些都在：
https://docs.docker.com/engine/install/ubuntu/
可以找到。

随后是踩坑的：

~~~shell
sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
~~~

`lsb_release -cs`查出来是`serena`，但是这个是Mint的Codename，需要查询对应的Ubuntu的版本：
在这里找：
https://www.linuxmint.com/download_all.php
我们找到是`xenial`，所以我们就

~~~shell
sudo add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    xenial \
    stable"
~~~

随后正常安装即可：

~~~shell
sudo apt-get update
sudo apt-get install docker-ce
sudo service docker start
sudo service docker status
~~~

