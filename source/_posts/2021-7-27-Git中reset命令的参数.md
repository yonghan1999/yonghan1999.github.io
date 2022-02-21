---
layout: post
title: Git中reset命令的参数
date: 2021-07-27
categories: 技术
tags: Git
---

### Git中reset的参数

今天使用Git的时候碰到的问题，使用reset命令相关参数不够了解，以下是相关参数介绍：

（1） 默认的mixed参数：`git reset commit_id`，将本地版本库的头指针全部重置到指定版本，且会重置暂存区，即这次提交之后的所有变更都移动到未暂存阶段。

（2） soft 参数：`git reset --soft commit_id` 意为将版本库软回退到`commit_id`指定的版本版本，所谓软回退表示将本地版本库的头指针全部重置到指定版本，且将这次提交之后的所有变更都移动到暂存区。

（3） hard参数：`git reset --hard commit_id` 意为将版本库回退`commit_id`指定的版本，但是不仅仅是将本地版本库的头指针全部重置到指定版本，也会重置暂存区，并且会将工作区代码也回退到这个版本 。
