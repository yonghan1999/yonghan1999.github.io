---
layout: post
title: LaTeX 学习过程
date: 2021-04-02
categories: 技术
tags: LaTeX
---

### LaTeX 的 hello word

~~~latex
\documentclass{article}
% 这里是导言区
\begin{document}
Hello, world!
\end{document}
~~~

- 以反斜杠 `\` 开头，以第一个**空格或非字母** 的字符结束的一串文字成为控制序列（或称命令/标记）
- % 是注释

### 宏包

通过```/usepackage{}```	可以调用宏包，下面是用 UTF-8 编码， XeLaTeX 编译的代码

~~~latex
\documentclass[UTF8]{ctexart}
\begin{document}
你好，world!
\end{document}
~~~

