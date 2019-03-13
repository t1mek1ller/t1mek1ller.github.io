---
layout: post
title: Scrapy架构
description: "Scrapy架构概述"
modified: 2014-06-09
tags: [Python, Scrapy]
categories: [Python, Scrapy]
---

之前利用scrapy做了个爬取医学问答数据的爬虫，对其实现很感兴趣，现在分析一下其整体设计架构，本文大部分翻译自[Architecture overview](http://doc.scrapy.org/en/latest/topics/architecture.html)

## 架构
下图展示了Scrapy整体架构及其各个组件，以及系统内部的数据流（用绿色箭头表示）

{% capture images %}
	/images/scrapy-architecture.png
{% endcapture %}
{% include gallery images=images caption="Scrapy架构" cols=1 %}

## 组件

### Scrapy引擎（Scrapy Engine）
引擎主要负责控制系统的数据流和各个组件，以及当特定的动作发生时触发相应的事件

### 调度器（Scheduler）
调度器负责收集从引擎发送过来的请求并将它们放入队列之中以待引擎的后续处理

### 下载器（Downloader）
下载器负责抓取web页面，并将它们交给引擎处理，最后再传给爬虫（spiders）

### 爬虫（[Spiders](http://doc.scrapy.org/en/latest/topics/spiders.html#topics-spiders)）
爬虫是一些由Scrapy用户自定义的类，用于分析返回页面并抽取数据项或是额外的urls。每一个spider有能力去处理一个特定的域名（或是一组域名）

### 数据项管道（[Item Pipeline](http://doc.scrapy.org/en/latest/topics/item-pipeline.html#topics-item-pipeline)）
数据项管道是负责处理由爬虫抽取的数据项。典型的任务包括清洗，验证，持久化等等。

### 下载器中间件（[Downloader middlewares](http://doc.scrapy.org/en/latest/topics/downloader-middleware.html#topics-downloader-middleware)）
爬虫中间件是一些介于引擎和爬虫之间的特定“钩子”，负责处理爬虫的输入（响应）和输出（数据项和请求）。它们提供了一个便利的机制以通过插入自定义代码来扩展Scrapy的功能

### 数据流（Data flow）
Scrapy中的数据流由执行引擎控制，整个数据流向如下：

1. 引擎打开一个域，定位处理该域的爬虫，并向爬虫询问第一个待爬取的URLs
2. 引擎从爬虫获取到URLs后，由调度器进行调度
3. 引擎向调度器询问下一个要爬取URLs
4. 调度器向引擎返回下一个URLs，引擎将它们发送给下载器，期间会经过下载器中间件的处理（请求方向）
5. 一旦页面下载完，下载器会生成响应并发送给引擎，期间会经过下载器中间件的处理（响应方向）
6. 引擎接收从下载器发来的响应，并将其发送给爬虫作进一步处理，期间会经过爬虫中间件的处理（输入方向）
7. 爬虫处理响应并返回数据项和新的请求给引擎
8. 引擎发送数据项（由爬虫返回）到数据项管道，发送新的请求（由爬虫返回）到调度器
9. 整个过程（从第二步开始）循环往复直到调度器没有新的下载请求，引擎关闭域

### 事件驱动网络（Event-driven networking）
Scrapy基于Python的一个流行的事件驱动网络框架[Twisted](http://twistedmatrix.com/trac/). 因此，通过实现非阻塞方式来进行并发控制。