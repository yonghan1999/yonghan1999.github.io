---
layout: post
title: Android Studio Gradle 设置代理
date: 2021-05-07
categories: 技术
tags: Android
---

### Android Studio Gradle 设置代理

~~~properties
systemProp.http.nonProxyHosts=*.xcompany.com|localhost
systemProp.http.proxyHost=127.0.0.1
systemProp.http.proxyPort=1087
systemProp.https.nonProxyHosts=*.xcompany.com|localhost
systemProp.https.proxyHost=127.0.0.1
systemProp.https.proxyPort=1087
~~~

这是AS自动配置的但是在我的 Mac 上只有这个配置是不能用的，需要在`grade.properties`中加入下面的配置，端口是本地socks5代理端口

~~~properties
org.gradle.jvmargs=-DsocksProxyHost=127.0.0.1 -DsocksProxyPort=1086 -DsocksNonProxyHosts=*.xcompany.com
~~~



