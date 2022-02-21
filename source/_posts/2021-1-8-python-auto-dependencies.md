---
layout: post
title: python-auto-dependencies
date: 2021-01-08
categories: 技术
tags: python
---

Python 自动生成当前项目依赖包文件

方法一

```python
# cd 到项目路径下，执行以下命令
pip freeze > requirements.txt
```

方法二

使用工具 pipreqs

```python
# 1 安装 pipreqs
pip install pipreqs

# 2 cd 到项目路径下，执行以下命令
pipreqs ./
```

使用 requests.txt 自动安装所有依赖包

```python
pip install -r requirements.txt
```