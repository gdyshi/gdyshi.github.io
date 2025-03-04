---
layout: post
title:  "windows下安装博客系统jekyll"
categories: JavaScript
tags:  博客 jekyll windows 环境搭建
---

* content
{:toc}

## 摘要
本文讲述如何在windows下搭建jeky11博客环境

## 引言
Jekyll 是一个简单的博客形态的静态站点生产机器。它有一个模版目录，其中包含原始文本格式的文档，
通过一个转换器（如 Markdown）和Liquid 渲染器转化成一个完整的可发布的静态网站，你可以发
布在任何你喜爱的服务器上。Jekyll 也可以运行在 GitHub Page 上，也就是说，你可以使用 GitHub 的
服务来搭建你的项目页面、博客或者网站，而且是完全免费的。为了方便地在发布博客之前看到博客效果，
需要在本地搭建Jekyll环境进行预览，本文对windows下搭建jeky11博客环境步骤进行记录

## 主题

### 安装 Ruby
* [下载](https://rubyinstaller.org/downloads/)
* 双击安装安装
    > 要安装在根目录，导入环境变量

* 测试 `ruby -v`

### 安装 DevKit
* [下载](https://rubyinstaller.org/downloads/)
* 解压到根目录
* 初始化 `ruby dk.rb init`
* 在`config.yml`的最后加上Ruby的安装路径
* 审查`ruby dk.rb review`及安装`ruby dk.rb install`

### 安装 Jekyll
```
gem -v

gem install jekyll
```

### 运行
```
cd myblog
jekyll s  / jekyll serve
```

## 附录


## 参考
---
- [Jekyllcn](http://jekyllcn.com/)