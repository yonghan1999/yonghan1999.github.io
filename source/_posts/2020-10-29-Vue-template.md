---
layout: post
title: Vue 踩坑
date: 2020-10-29
categories: 技术
tags: Vue
---

又要写一学期一度的大作业，这次试试用下 Vue， 下面是踩坑集锦 (— —!).

### Component template should contain exactly one root element. If you are using v-if on multiple elements, use v-else-if to chain them instead.

第一次写 template 试了一下 ElementUI 的一个样例，直接粘贴到 template 内报错了

~~~html
<template>
  <el-row>
    <el-col :span="24"><div class="grid-content bg-purple-dark"></div></el-col>
  </el-row>
  <el-row>
    <el-col :span="12"><div class="grid-content bg-purple"></div></el-col>
    <el-col :span="12"><div class="grid-content bg-purple-light"></div></el-col>
  </el-row>
  <el-row>
    <el-col :span="8"><div class="grid-content bg-purple"></div></el-col>
    <el-col :span="8"><div class="grid-content bg-purple-light"></div></el-col>
    <el-col :span="8"><div class="grid-content bg-purple"></div></el-col>
  </el-row>
  <el-row>
    <el-col :span="6"><div class="grid-content bg-purple"></div></el-col>
    <el-col :span="6"><div class="grid-content bg-purple-light"></div></el-col>
    <el-col :span="6"><div class="grid-content bg-purple"></div></el-col>
    <el-col :span="6"><div class="grid-content bg-purple-light"></div></el-col>
  </el-row>
  <el-row>
    <el-col :span="4"><div class="grid-content bg-purple"></div></el-col>
    <el-col :span="4"><div class="grid-content bg-purple-light"></div></el-col>
    <el-col :span="4"><div class="grid-content bg-purple"></div></el-col>
    <el-col :span="4"><div class="grid-content bg-purple-light"></div></el-col>
    <el-col :span="4"><div class="grid-content bg-purple"></div></el-col>
    <el-col :span="4"><div class="grid-content bg-purple-light"></div></el-col>
  </el-row>
</template>
~~~

基础不牢的菜鸟翻了好一会儿文档才找到，原来 **Vue 模板只能有一个跟对象**。

所以你想要出现正常的效果，你的用一个div来或是别的标签来包裹全部的元素。

正确的写法就是：

~~~html
<template>
	<div>
		<el-row>
		  <el-col :span="24"><div class="grid-content bg-purple-dark"></div></el-col>
		</el-row>
		<el-row>
		  <el-col :span="12"><div class="grid-content bg-purple"></div></el-col>
		  <el-col :span="12"><div class="grid-content bg-purple-light"></div></el-col>
		</el-row>
		<el-row>
		  <el-col :span="8"><div class="grid-content bg-purple"></div></el-col>
		  <el-col :span="8"><div class="grid-content bg-purple-light"></div></el-col>
		  <el-col :span="8"><div class="grid-content bg-purple"></div></el-col>
		</el-row>
		<el-row>
		  <el-col :span="6"><div class="grid-content bg-purple"></div></el-col>
		  <el-col :span="6"><div class="grid-content bg-purple-light"></div></el-col>
		  <el-col :span="6"><div class="grid-content bg-purple"></div></el-col>
		  <el-col :span="6"><div class="grid-content bg-purple-light"></div></el-col>
		</el-row>
		<el-row>
		  <el-col :span="4"><div class="grid-content bg-purple"></div></el-col>
		  <el-col :span="4"><div class="grid-content bg-purple-light"></div></el-col>
		  <el-col :span="4"><div class="grid-content bg-purple"></div></el-col>
		  <el-col :span="4"><div class="grid-content bg-purple-light"></div></el-col>
		  <el-col :span="4"><div class="grid-content bg-purple"></div></el-col>
		  <el-col :span="4"><div class="grid-content bg-purple-light"></div></el-col>
		</el-row>
	</div>
</template>
~~~

### 配置路径别名

~~~javascript
// vue.config.js
module.exports = {
  "transpileDependencies": [
    "vuetify"
  ]
    
}
~~~

