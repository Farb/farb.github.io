---
title: 鸿蒙界面开发--美团购物车
description:
slug: harmony-meituancart
date: 2024-04-10 00:00:00+0000
image: 
categories:
    - HarmonyOS
tags:
    - ArkTs
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 界面效果
![](MeiTuanCart.png)

## 代码

``` ts
@Entry
@Component
struct MeiTuanCartPage {
  @State oldPrice: number = 40.4
  @State newPrice: number = 20
  @State buyCount: number = 1

  build() {
    Column() {
      Row() {
        Image($r('app.media.mt_product1'))
          .width(100)
          .borderRadius(10)

        Column({ space: 10 }) {
          Column({ space: 6 }) {
            Text('冲销量1000ml缤纷八果水果捞')
              .maxLines(2)
              .textOverflow({ overflow: TextOverflow.Ellipsis })
              .fontWeight(FontWeight.Bold)
            Text('含一份折扣商品')
          }
          .alignItems(HorizontalAlign.Start)

          Row() {
            Row({ space: 10 }) {
              Text('￥' + this.newPrice)
                .fontColor(Color.Red)
                .fontWeight(FontWeight.Bolder)
                .fontSize(20)
              Text('￥' + this.oldPrice)
                .fontColor('#666')
                .decoration({
                  type: TextDecorationType.LineThrough
                })
            }

            Row() {
              Text('-')
                .width(20)
                .textAlign(TextAlign.Center)
                .fontWeight(FontWeight.Bold)
                .border({
                  width: 1,
                  color: '#666',
                  radius: {
                    topLeft: 5,
                    bottomLeft: 5
                  }
                })
                .onClick(() => {
                  if (this.buyCount > 1) {
                    this.buyCount--
                  }
                })
              Text(this.buyCount.toString())
                .width(30)
                .textAlign(TextAlign.Center)
                .border({
                  width: 1,
                  color: '#666'
                })
              Text('+')
                .width(20)
                .textAlign(TextAlign.Center)
                .fontWeight(FontWeight.Bold)
                .border({
                  width: 1,
                  color: '#666',
                  radius: {
                    topRight: 5,
                    bottomRight: 5
                  }
                })
                .onClick(() => {
                  if (this.buyCount < 100) {
                    this.buyCount++
                  }
                })
            }.padding({
              right: 20
            })
          }
          .justifyContent(FlexAlign.SpaceBetween)
          .width('100%')
          .height(30)
        }
        .alignItems(HorizontalAlign.Start)
        .padding({
          left: 10
        })
        .layoutWeight(1)
      }
      .width('100%')

      Row() {
        Blank()
        Column({ space: 10 }) {
          Text() {
            Span(`已选${this.buyCount}件，合计`)
            Span(`￥${(this.buyCount * this.newPrice).toFixed(2)}`)
              .fontColor(Color.Red)
          }

          Text('共减￥' + ((this.oldPrice - this.newPrice) * this.buyCount).toFixed(2))
            .fontColor(Color.Red)
        }
        .alignItems(HorizontalAlign.End)

        Button('结算')
          .backgroundColor('#f5da46')
          .fontColor(Color.Black)
          .fontSize(20)
          .padding(10)
          .width(120)
          .margin({
            left: 15
          })
      }
      .padding(15)
      .width('100%')
    }
    .padding(10)
    .width('100%')
    .height('100%')
  }

  @Builder NumberInput() {
    Row() {
      Text('-')
        .border({
          width: 1,
          color: '#666',
          radius: {
            topLeft: 5,
            bottomLeft: 5
          }
        })
        .onClick(() => {
          this.buyCount--
        })
      Text(this.buyCount.toString())
      Text('+')
        .onClick(() => {
          this.buyCount++
        })
    }
    .layoutWeight(1)
    .justifyContent(FlexAlign.SpaceBetween)
    .margin({ right: 5 })
  }
}
```