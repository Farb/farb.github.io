---
title: "Harmonyos_ui_customdialog"
description: 
slug: arkts-customdialog
date: 2024-04-13T22:47:26+08:00
image: 
categories:
    - HarmonyOS
tags:
    - ArkTs
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## ArkTs自定义弹框

效果图：


## 实现代码

``` ts
// 一个内容可以滚动的单选列表
@Entry
@Component
struct CustomDialogPage {
  @State message: string = 'Hello World'

  build() {
    Row() {
      Column() {
        CustomDialogExample()
      }
      .width('100%')
    }
    .height('100%')
  }
}

//@CustomDialog
@Component
struct CustomDialogExample {
  @State students: Student[] = []
  // controller: CustomDialogController
  // cancel: () => void
  // confirm: () => void

  aboutToAppear() {
    this.students = this.loadData()
  }

  loadData(): Array<Student> {
    let names: string[] = ['小明', '小红', '小刚', '小美', '小杜', '小帅', '小李', '小王', '小刘', '小杨', '小宋','Farb']
    const data: Student[] = []
    let i: number
    for (i = 0; i < names.length; i++) {
      data.push(new Student(i + 1, names[i]))
    }
    return data
  }

  build() {
    Column() {
      Text('请选择一个学生').fontSize(20).margin({ top: 10, bottom: 10 })
      TextInput({ placeholder: '请输入学生姓名进行搜索' })
        .placeholderColor('#ccc')
        .onChange(value => {
          if (value) {
            this.students = this.loadData().filter(item => item.name.indexOf(value) >= 0)
          } else {
            this.students = this.loadData()
          }
        })
      Scroll() {
        Column() {
          ForEach(this.students, (student: Student) => {
            StudentCard({ student: student })
          })
        }
      }.scrollBar(BarState.Off)

      Button('取消')
        .fontSize(18)
        .fontWeight(FontWeight.Bold)
        .fontColor('#3CBEF5')
        .backgroundColor(Color.White)
    }
    .height(300)
  }
}

@Component
struct StudentCard {
  @ObjectLink student: Student

  build() {
    Row() {
      Text(this.student.name)
        .layoutWeight(1)
      Radio({ value: '', group: 'default' })
        .checked(this.student.isChecked)
        .onChange(isChecked => {
          // 每个单选按钮状态的变化都要同步到student属性isChecked上
          this.student.isChecked = isChecked
        })
    }
    .padding(10)
    .justifyContent(FlexAlign.SpaceBetween)
    .onClick(() => {
      this.student.isChecked = true
    })
    .width('30%')
  }
}


@Observed
class Student {
  id: number
  name: string
  isChecked: boolean

  constructor(id: number, name: string, isChecked?: boolean) {
    this.id = id
    this.name = name
    this.isChecked = isChecked
  }
}

```