---
layout: post
title: 关于centos上配置https总结
date: 2019-08-18
categories: 技术
tags: linux
---

## 关于centos上配置https总结

购买到免费的证书之后，下载证书（因为我们使用的是Apache所以在下载的时候选择下载适用于Apache的证书）

										![](域名购买截图-768x461.png)											

购买到免费的证书之后，下载证书（因为我们使用的是Apache所以在下载的时候选择下载适用于Apache的证书）

										![](域名下载截图.png)											

确保服务器安装了：openssl和openssl-devel，httpd-devel；没有安装的话用yum安装一下就好了

`yum install openssl  

yum install openssl-devel  

yum install httpd-devel  

`**默认apache是没有安装SSL模块的，所以需要安装  
**`yum install -y mod_ssl`然后去配置Apache的配置文件httpd.conf  

`LoadModule ssl_module modules/mod_ssl.so  #去掉前面的注释，没有的话就自己加上去加载ssl模块，去modules文件夹下面能找到mod_ssl.so文件，如果没有那就是mod_ssl模块没有安装正确  

Include conf.d/httpd-ssl.conf   #去掉注释，
`

然后去配置ssl.conf

										![](ssl_conf大致.png)											

做完这些记得要开放https的默认端口443端口，然后重启Apache服务器就完成https的配置了  
`systemctl restart httpd.service`

等待1~2分钟就可以用https访问了。