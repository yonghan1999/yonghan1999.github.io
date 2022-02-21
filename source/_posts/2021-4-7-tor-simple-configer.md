---
layout: post
title: tor 的简单配置
date: 2021-04-07
categories: 技术
tags: Network
---

### Tor 是什么？

Tor（The Onion Router，洋葱路由器）是实现匿名通信的自由软件。Tor是第二代洋葱路由的一种实现，用户通过Tor可以在因特网上进行匿名交流。
而对于爬虫爱好者来说，Tor会隐藏你的真实ip地址，导致被爬网站的网管无法封掉你的ip（当然是在对方觉得精确定位你的ip无价值的情况下）安全是相对的，匿名也是相对的。

### 安装 Tor

如果你用的是 MacOS 并且安装了 Homebrew ，那么你可以通过下面的命令轻松的安装 Tor

~~~bash
brew install tor
~~~

或者去 https://www.torproject.org/ 寻求其他系统的版本。

### 简单配置

如果你用 Homebrew 安装完成之后，那么你将在`/opt/homebrew/etc/tor`路径下找到`torrc.sample`文件，这是一个 Tor 的配置文件模版，我们需要的是删除 `.simple`后缀，然后在文件中添加以下内容：

~~~python
CookieAuthentication 1

# 代理地址为 127.0.0.1:1087
# HTTPProxy 127.0.0.1:1087
# HTTPSProxy 127.0.0.1:1087
Socks5Proxy 127.0.0.1:1086
# 外部程序Tor 的端口
SocksPort 9050
# 自动切换 ip 的时间
MaxCircuitDirtiness 10 seconds
~~~

### 运行

你可以直接运行`Tor`命令启动 Tor

~~~
# tor
Apr 07 16:21:16.485 [notice] Tor 0.4.5.7 running on Darwin with Libevent 2.1.12-stable, OpenSSL 1.1.1k, Zlib 1.2.11, Liblzma N/A, Libzstd N/A and Unknown N/A as libc.
Apr 07 16:21:16.485 [notice] Tor can't help you if you use it wrong! Learn how to be safe at https://www.torproject.org/download/download#warning
Apr 07 16:21:16.486 [notice] Read configuration file "/opt/homebrew/etc/tor/torrc".
Apr 07 16:21:16.489 [notice] Opening Socks listener on 127.0.0.1:9050
Apr 07 16:21:16.490 [notice] Opened Socks listener connection (ready) on 127.0.0.1:9050
Apr 07 16:21:16.000 [notice] Parsing GEOIP IPv4 file /opt/homebrew/Cellar/tor/0.4.5.7/share/tor/geoip.
Apr 07 16:21:16.000 [notice] Parsing GEOIP IPv6 file /opt/homebrew/Cellar/tor/0.4.5.7/share/tor/geoip6.
Apr 07 16:21:16.000 [notice] Bootstrapped 0% (starting): Starting
Apr 07 16:21:16.000 [notice] Starting with guard context "default"
Apr 07 16:21:17.000 [notice] Bootstrapped 3% (conn_proxy): Connecting to proxy
Apr 07 16:21:17.000 [notice] Bootstrapped 4% (conn_done_proxy): Connected to proxy
Apr 07 16:21:17.000 [notice] Bootstrapped 10% (conn_done): Connected to a relay
Apr 07 16:21:18.000 [notice] Bootstrapped 14% (handshake): Handshaking with a relay
Apr 07 16:21:19.000 [notice] Bootstrapped 15% (handshake_done): Handshake with a relay done
Apr 07 16:21:19.000 [notice] Bootstrapped 75% (enough_dirinfo): Loaded enough directory info to build circuits
Apr 07 16:21:19.000 [notice] Bootstrapped 90% (ap_handshake_done): Handshake finished with a relay to build circuits
Apr 07 16:21:19.000 [notice] Bootstrapped 95% (circuit_create): Establishing a Tor circuit
Apr 07 16:21:20.000 [notice] Bootstrapped 100% (done): Done
~~~

MacOS 中你也可以通过`brew services start tor`来让 Tor 在后台运行。