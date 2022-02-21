---
layout: post
title: proxychais 无法代理 ping 命令的原因
date: 2021-08-05
categories: 技术
tags: 网络小工具 
---
### proxychains无法代理ping命令的原因

proxychains支持的是socks、http和https协议，它们是以tcp或者udp为基础的，所以proxychains只支持使用tcp或udp协议的程序。然而ping命令使用的是ICMP协议，自然无法使用。