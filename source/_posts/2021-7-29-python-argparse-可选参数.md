---
layout: post
title: python3 argparse 可选参数
date: 2021-07-29
categories: 技术
tags: python
---

## argparse 可选参数

如何使用`argparse`的`add_argument()`函数，以便用户必须解析一个必需的值，也可能解析一个可选值？在

例如`--read book [page]`。您可以省略`page`，也可以解析要阅读的特定页面。

在调用中添加`nargs='?'`，并将值1作为默认值（也可能将`type=int`解析为数字）：

```python
parser.add_argument(' read', dest='book', help='book to read')
parser.add_argument('page', nargs='?', default=1, type=int, help='page number')
```

