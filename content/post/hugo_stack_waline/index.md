---
title: Hugo+Stack主题快速集成评论系统Waline
description: Waline 功能相对丰富点
slug: hugo_stack_waline
date: 2024-04-21 
image: 
categories:
    - Blog
tags:
    - Hugo Stack
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 实验经历

之前集成了utterance，但是它必须登录才能发布评论，而且博客发布到国内的gitee上的话，无法展示。所以今天换成了Waline。

**[Waline官网](https://waline.js.org/)**

## 参考文档

https://waline.js.org/guide/get-started/

跟着Waline官方这个文档一步步来，就可以。主要思想就是部署一个数据库和nodejs的服务端。数据库部署在LeanCloud,服务端部署在Vercel上。
**需要注意的是，Vercel上部署的应用，需要绑定自己的域名才可以，据说是DNS污染，因此买了个域名farb.top。**

## Hugo Stack中如何配置
1. 找到配置文件： config\_default\params.toml
2. 修改评论系统的provider以及`[comments.waline]`节点下的`serverURL`为你自己的后端服务地址。代码如下：
   ``` toml
    ## Comments
    [comments]
    enabled = true
    provider = "waline"   ## 修改点

   [comments.waline]
    serverURL = "https://blog.comment.farb.top"  ## 修改点
    lang = "zh-cn"  ## 修改点
    visitor = ""
    avatar = ""
    emoji = ["https://cdn.jsdelivr.net/gh/walinejs/emojis/weibo"]
    requiredMeta = ["name", "email"]
    placeholder = ""
   ```

## 效果展示

![](https://s3.bmp.ovh/imgs/2024/04/21/4532be3fa12f4f52.png)