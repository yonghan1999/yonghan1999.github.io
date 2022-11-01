---
layout: post
title: Flutter console
date: 2022-11-01
categories: 技术
tags: Flutter
---

记录我刚开始初学 Flutter 并使用 IntelliJ IDEA 作为开发的IDE，我想将数据记录到控制台？我试过`print()`and `printDebug()`，但我的数据都没有显示在 Flutter 控制台中。

后面我查看了一些资料知道，如果你在 Flutter 里面`Widget`，你可以使用[`debugPrint`](https://api.flutter.dev/flutter/foundation/debugPrint.html)，例如，

```dart
import 'package:flutter/foundation.dart';

debugPrint('movieTitle: $movieTitle');
```

或者，使用 Dart 的内置`log()`函数

```dart
import 'dart:developer';

log('data: $data');
```

> Dart **print()**函数输出到系统控制台，可以使用 Flutter 日志（基本上是 adb logcat 的包装器）查看它。
>
> 如果一次输出太多，那么 Android 有时会丢弃一些日志。为避免这种情况，可以使用**debugPrint()**。
>
> 在这里找到：[https ://flutter.io/docs/testing/debugging](https://flutter.io/docs/testing/debugging)