---
layout: post
title: Upgrading NodeJS causes Sass compilation error
date: 2021-04-05
categories: 技术
tags: NodeJS
---

### MacOS 更换 NodeJS 后 Sass 报错

~~~bash
Syntax Error: Error: Node Sass does not yet support your current environment: OS X Unsupported architecture (arm64) with Unsupported runtime (88)
~~~

![node-error.png](https://i.loli.net/2021/04/05/Qy8MtvPDGNcKior.png)

这个错误是我在切换 NodeJS 版本的时候遇到的（从 x86 切换至 arm）

错误的解释建议阅读此发行信息：[https](https://github.com/sass/node-sass/releases/tag/v4.5.0) : [//github.com/sass/node-sass/releases/tag/v4.5.0](https://github.com/sass/node-sass/releases/tag/v4.5.0)。我认为这不是很有用。

我在论坛上阅读了一些答案，例如删除 `node_modules` 目录并通过 `npm install` 重新安装所有节点软件包。**不要那样做！**如果您在项目的开发阶段，如果 package.json 的软件包版本不干净，您可能会遇到麻烦。

### 如何快速简单的修复？

我不能解释当我们切换 NodeJS 版本的时候发生了什么，但我知道可以用以下命令修复该错误

~~~bash
npm rebuild node-sass
~~~

这可能需要一些时间，不用担心。

