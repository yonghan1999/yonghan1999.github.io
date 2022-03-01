---
layout: post
title: Flutter BLoC
date: 2022-03-01
categories: 技术
tags: Flutter
---

最近在又又又开始重构我的Flutter项目😇，继上次堆代码把功能实现了然后代码越写越复杂耦合越来越严重💀，于是我决定短痛，推了重写（这次我一定设计（chao）一个好的架构并维护好它💪）。先从业务逻辑和UI组件的结偶下手，决定用这个👉BLoC (**B**usiness **L**ogic **C**omponents) 模式，BLoC的哲学就是app里的所有东西都应该被认为是事件流：一部分组件订阅事件，另一部分组件则响应事件。BLoC居中管理这些会话。Dart甚至把流（Stream）内置到了语言本身里。这个模式最好的地方就是你不需要引入任何的插件，也不需要学习其他的语法。所有需要的内容Flutter都有提供。

PS：学起来有点难 😭。　

[![streams_bloc.png](https://image.hanblog.fun/images/2022/03/01/streams_bloc.png)](https://image.hanblog.fun/image/1G1)

- *小部件通过Sinks*向 BLoC发送*事件*。
- BLoC 通过*流*通知小部件。
- BLoC 实现的业务逻辑与他们无关。

这样可以使业务逻辑与 UI 的解耦，这样我么就可以：

- 我们可以随时更改业务逻辑，对应用程序的影响最小，
- 我们可以在不影响业务逻辑的情况下更改 UI，
- 现在测试业务逻辑要容易得多。

同时现在也已经有非常受欢迎的Flutter_bloc来支持你实现这个模式：

[<img src="https://image.hanblog.fun/images/2022/02/27/bloc_architecture_full.png" alt="bloc_architecture_full.png" style="zoom:67%;" />](https://image.hanblog.fun/image/S7V)

[<img src="https://image.hanblog.fun/images/2022/02/27/bloc_flow.png" alt="bloc_flow.png" style="zoom:67%;" />](https://image.hanblog.fun/image/aze)

[文档在这里](https://bloclibrary.dev/#/gettingstarted)
