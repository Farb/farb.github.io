---
title: 鸿蒙界面开发--常见问题
description:
slug: harmony-faq
date: 2024-04-22
image: 
categories:
    - HarmonyOS
tags:
    - ArkTs
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 1. 使用svg图片填充颜色fillColor时，不生效
原因可能是其他网站上的svg格式和华为提供的格式有区别。解决方案如下：

![](https://s3.bmp.ovh/imgs/2024/04/22/bcbc3d2c04d6ba3f.png)

## 2. Text的子组件Span使用时注意点
 1. Text和Span都有内容时，Text的内容会被Span覆盖
 2. Text的赋予属性（如字体大小和颜色）时，Span会继承，如果Span有一个属性重新赋值，则其他属性也不会继承。
![](https://s3.bmp.ovh/imgs/2024/04/22/1a9e908a5092eefc.png)