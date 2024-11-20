---
layout: post
title: Typora自动上传图片到Chevereto
date: 2024-11-20
categories: 工具
tags: 工具 Typora Chevereto
---

登录chevereto服务器，`右上角账号 -- 仪表盘 -- 设置 -- API`

![image-20220618085157380](https://image.hanblog.fun/images/2024/11/20/3a71e2dec4f2c263db58cda6d2dcecc2.png)

复制一下这里的`API v1 密钥`

![QQ截图20220618085235](https://image.hanblog.fun/images/2024/11/20/3c9d7560e9812bb9c7d204073ccd41a6.png)

使用`pf`工具（工具链接：[https://github.com/yonghan1999/picture_fly/releases/latest](https://github.com/yonghan1999/picture_fly/releases/latest)），下载自己所需环境的包。并将工具配置到PATH环境变量下。

~~~bash
# 命令介绍
$ pf.exe --help

A simple chevereto image upload tool.

Usage: pf.exe [OPTIONS] <KEY> <URL> [FILES]...

Arguments:
  <KEY>       chevereto api key
  <URL>       your chevereto upload url e.g. https://images.hanblog.fun/api/1/upload/
  [FILES]...  image file path. e.g. https://image.hanblog.fun/images/2024/11/20/test.jpg OR local file path

Options:
  -f, --force
  -p, --print
  -h, --help     Print help
  -V, --version  Print version
~~~

接下来我们打开Typora的设置配置上传图像。

![image-20241120211111325](https://image.hanblog.fun/images/2024/11/20/image-20241120211111325.png)

命令的地方填写`pf <KEY> <URL>`，这里的`<Key>`就是前面我们赋值出来的`API v1 密钥`，`<URL>`填写图片的上传地址Chevereto一般是` https://${domain}/api/1/upload/`，例如我的是` https://image.hanblog.fun/api/1/upload/`。

例如我的命令就是`pf thisismyapikey https://image.hanblog.fun/api/1/upload/ `。然后我们点击`验证图片上传选项`按钮检测是否能个正确上传图片。

![image-20241120211905801](https://image.hanblog.fun/images/2024/11/20/image-20241120211905801.png)



