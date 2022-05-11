---
layout: post
title: Dart异步Stream和Future
date: 2022-05-11
categories: 技术
tags: Dart
---

前言：初学`Flutter`时，是因为被她的全平台和性能所吸引，没什么耐心的我粗略的看了`dart`的新手guide简单的看了`Flutter`的例子之后，感觉`Dart`语法和`Java`或者说`Javascript`差别不是很大，当时的我觉得并没有什么难度，异步编程也一知半解，看了Future之后就没继续看下去了。结果给自己埋了一个大坑（。

这里我简单介绍一下我对[**Future**](https://api.dart.dev/stable/2.12.4/dart-async/Future-class.html)和[**Stream**](https://api.dart.dev/stable/2.12.4/dart-async/Stream-class.html)是Dart语言处理非同步事件的二大核心类别，你是否觉得二者很容易搞混，或是对它俩一知半解呢？其实只要一个简单的观念你马上就会觉得「喔~~原来是这样！」，并且永远不会再搞错了！

> 有了这两种异步之后，就可以解决Flutter中生命周期中一些页面数据更新的问题了!

其实官方对他们的定义就非常准确了

**Future**

> A Future represents a computation that doesn't complete immediately. Where a normal function returns the result, an asynchronous function returns a Future, which will eventually contain the result. The future will tell you when the result is ready.

Future代表一个不会立即完成的计算过程，不同于一般函式回传一个结果，一个非同步函式是回传一个*Future*，你会一直等待到Future执行完成后才会取得最后结果。

**Stream**

> A stream is a sequence of asynchronous events. It is like an asynchronous Iterable — where, instead of getting the next event when you ask for it, the stream tells you that there is an event when it is ready.

Stream是一个非同步的事件序列，它就如同一个非同步的循环，你可能无法在向Stream取得事件的当下取得资料，只要你尚未停止「取得事件」的状态，你将会在最新事件准备好时通知你。

如果你还是不明白，我给你打个比方：

现在有一家餐厅，他会制作出食物（数据），Future获得到食物就类似于你订外卖（或者到店取餐），就是等到食物做好之后，他会一次性给你。而Stream则是你到店里吃，服务员可以做完一道菜之后给你上一道菜，这是一个持续性的过程。