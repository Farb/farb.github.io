---
title: Git rebase 命令行操作
description:
slug: git_rebase
date: 2024-04-25
image: 
categories:
    - Git
tags:
    - Git
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 1. Git rebase 和 Git merge
git merge相比git rebase使用更简单，网上对于这两种git工作流的看法也是两个流派。
有人喜欢merge , 有人喜欢rebase。聪明的人不会做偏执狂，团队中让用哪种，咱就用那种就对了，都了解一下没啥坏处。

在宏观上，git rebase生成的线条和节点更简单清晰，git merge因为会自动合并，当团队成员较多时，产生的线条和节点不太清晰。
微观上，git rebase会创建提交的副本让提交历史更加线性化，git merge会创建自动合并的提交。

点击下面的链接详细了解两种工作流。

https://www.cnblogs.com/FraserYu/p/11192840.html

https://zhuanlan.zhihu.com/p/686538265

这里推荐一个动画学习git的网站：https://learngitbranching.js.org/?locale=zh_CN

## 2. Git rebase 命令行

``` bash
git remote -v  # 查看远程仓库，如果没有origin仓库需要加一下
git remote add upstream remoteUrl # 将远程仓库加到上游
git remote -v # 再次查看远程仓库
git branch -r # 查看远程仓库的分支

# 以下几个命令需要经常使用
git remote update upstream # 更新上游仓库origin代码

git rebase upstream/master # rebase上游仓库master分支的代码到当前分支
# 如果有冲突，需要解决，解决之后git rebase --continue ,然后再推送

# 必须强制推送，因为本地合并的代码已经是确定要保留的代码，如果没有强制推送，则会提示先pull，如果真的pull的话，会导致代码混乱
git push -f  
```