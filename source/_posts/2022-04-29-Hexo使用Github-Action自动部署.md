---
layout: post
title: Hexo使用Github Action自动部署
date: 2022-04-29
categories: 技术
tags: Github
---

使用Hexo搭建博客后，用 Github Action 自动部署。

> 再见👋 NodeJS，\**** u。

在你的 Github 仓库中点击 `Action` 按钮，如果你从来都没有创建过`Action`，那么再点击`I understand my workflows, go ahead and enable them`按钮，然后复制下面的配置。去看Github的最近你添加Action的那次提交是否正常执行，相信你一定幸运的得到了自己想要的结果。如果你不够幸运，那就点❌看看报错信息，相信最终你会成功。

> https://github.com/features/actions 它可能会有帮助

~~~yml
name: Pages

on:
  push:
    branches:
      - master  # 要编译的分支

jobs:
  pages:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: recursive
      - name: Use Node.js 17.x
        uses: actions/setup-node@v2
        with:
          node-version: '17'   # NodeJs的版本根据你本地的版本写
      - name: Cache NPM dependencies
        uses: actions/cache@v2
        with:
          path: node_modules
          key: ${{ runner.OS }}-npm-cache
          restore-keys: |
            ${{ runner.OS }}-npm-cache
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./pub # 要发布的目录，也可能叫 publish，
~~~

