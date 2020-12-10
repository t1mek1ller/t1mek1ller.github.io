---
layout: post
title: 利用Jekyll和Github搭建博客
description: "利用Jekyll和Github搭建博客"
modified: 2014-04-15
tags: [Jekyll, Github]
categories: [Blog]
---

目前的博客系统多的数不胜数，眼花缭乱。用的最多的wordpress虽然方便，但总觉得过于臃肿不堪，关键是自己找免费的服务器(sae, bae, openshift, etc.)很是麻烦，自己的博客数据也不可控制。
网上浏览博客时发现很多基于[Jekyll](http://jekyllrb.com/)的博客，并且搭建在[github](http://github.com/)上。

* Jekyll是所谓的静态博客系统，把文章都变成文本形式存储，而不用数据库，非常轻便。
* Github就不用说了，基于Git的版本控制网站，提供免费的博客空间

这种组合优点在于：

* 自己的博客数据可以通过git随时随地更新和存取，可以版本控制，对于程序员来说真是极好的
* 各种主题够炫酷，也可以自己修改
* 还不用付钱

遂尝试之。

网上的教程千篇一律，令人头疼，其实最有用的还是官方教程，我参考的资料如下:

* [Github博客帮助页面](https://help.github.com/categories/github-pages-basics/)，建立repository用的
* [Jekyll主题](http://jekyllthemes.org/)，每个主题都会有个theme setup教程，非常实用
* [Jekyll官方网站](http://jekyllrb.com/)，可以弄清楚博客内部各个文件夹的作用

下面给出一个简单快速的建立过程：

## github新建repository
名称与用户名相同，如username.github.io，github提供自动生成主页的功能（在repository的setting中设置），这样就可以通过 **http://username.github.io** 访问了，但是没什么用，只有一个主页。可以通过git把这个项目clone本地，在本地修改上传，之后写文章等都是通过git上传。

## Jekyll安装
github帮助手册[Using Jekyll with Pages](https://help.github.com/articles/using-jekyll-with-pages/)已经说的很清楚。其中重点如下：

* 安装ruby，版本>=1.9.3, 我的是ubuntu12.04LTS系统，默认的是1.8的，所以需要自己升级一下版本
* 安装bundler，这是一个包管理软件，可以维护jekyll的更新，通过命令 **gem install bundler** 安装 
* 安装Jekyll，需要在自己的repository中新建一个Gemfile文件（一般主题之中Gemfile，此处可以先添上两行**source 'https://rubygems.org'**，**gem 'github-pages'**），然后执行**bundle install**

此时如果到repository目录执行**bundle exec jekyll serve**命令，就可以通过**http://localhost:4000**预览自己的主页了。如果自己不修改主题的话，其实也没必要装，之后写文章在windows下写用git推送就OK了。

## 安装Jekyll主题
从[Jekyll主题](http://jekyllthemes.org/)中挑选一个自己喜欢的，可以看看demo，demo中一般都有theme setup教程，一般流程是从该主题的github主页中fork其项目，然后clone到本地，将其中的文件夹拷贝到自己的repository中（不要拷贝.git文件夹，因为自己的repository有了），然后同样执行**bundle exec jekyll serve**就可以达到主题demo的效果:)

## Markdown写文章
终于可以写文章了，那如何写呢？一般上面安装的主题下都有一个**_post**的文件夹，写的文章都放里面了，可以通过**markdown**编写，文件名格式为**年-月-日-文章名.md**，文件内容兼容**markdown**语法，具体的也可以参照主题中的post样本。

我是用[sublime text 3](http://www.sublimetext.com/3)写的，利用其**package control**可以下载诸多插件，我下载了**markdown preview**用来编写和预览文件，另外还推荐一个**evernote**插件，可以和evernote双向存取，挺好用的。

