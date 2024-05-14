---
title: 鸿蒙界面开发--常见问题
description:
slug: harmony-ui-marquee
date: 2024-05-15
image: 
categories:
    - HarmonyOS
tags:
    - ArkTs
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 1. 需求描述

显示一个字符串，**后面紧跟着一个图标**。
当字符串较短时，正常显示文本，当字符串较长（超过屏幕宽度减去图片宽度和元素的间距总和）时，显示跑马灯效果。


## 2. 代码展示

``` ts
@Entry
@Component
struct MarqueeDemo {
  @State text: string = '这是一个短字符串'
  @State isShowMarquee: boolean = false

  build() {
    Column() {
      Row() {
        if (this.isShowMarquee) {
          Marquee({
            start: true,
            src: this.text
          })
            .width(300)
        } else {
          Text(this.text)
            .fontSize(14)
            .onAreaChange((oldValue, newValue) => {
              if (newValue.width > 280) {
                this.isShowMarquee = true
              }
              console.log('ooo', JSON.stringify(oldValue))
              console.log('nnn', JSON.stringify(newValue))
            }
            ).margin({
            left: 10
          })
        }
        Image($r('app.media.product'))
          .width(20)
          .margin(10)
      }

      Button('切换字符串长度')
        .onClick(() => {
          this.text = '这是一个很长的字符串，目的是为了测试跑马灯效果'
        })
    }
    .width('100%')
  }
}
```

## 3. 效果呈现

![](https://s3.bmp.ovh/imgs/2024/05/15/eb5169a9b23fbb09.gif)