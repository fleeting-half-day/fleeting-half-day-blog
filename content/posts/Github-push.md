---
title: "本地项目上传 Github"
date: 2024-09-18
metaAlignment: center
categories:
- Git 使用
tags:
- Git
---

<!--more-->

### 本地项目上传 Github

#### 创建本地库
*进入到项目根目录，初始化本地仓库*
```shell
git init
git add .
git commit -m 'first commit'
# 查看分支
git branch
# 本地默认分支是 main，github 远程仓库默认分支 master
# 更改本地分支名
git branch -m main master
```
#### 创建 github 远程分支
*创建远程仓库不作详细介绍，注意本地分支和远程分支保持一致。*
#### 关联远程仓库
```shell
git remote add origin https://github.com/expo/demo.git
```
#### 本地仓库到远程仓库
```shell
# 第一次推送加 -u 参数，添加上游（跟踪）引用，之后推送不再需要
git push -u origin master
```
*如果本地项目过大，可能会报错：*
> error: RPC failed; HTTP 400 curl 22 The requested URL returned error: 400
> send-pack: unexpected disconnect while reading sideband packet
*这是因为git默认缓存区太小，数据没有推送完成就断开了，所以加大缓存区可以解决：*
```shell
# 缓存区设置为：524288000B（500M）
git config http.postBuffer 524288000
```
