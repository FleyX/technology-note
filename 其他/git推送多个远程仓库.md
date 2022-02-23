---
id: "20220223"
date: "2022/02/23 10:38:05"
title: "git配置多个远程地址"
tags: ["git", "私有仓库", "gitlab", "gogs"]
hide: false
index_img: https://qiniupic.fleyx.com/blog/202202231638624.png?imageView2/2/w/200
banner_img: https://qiniupic.fleyx.com/blog/202202231638624.png
categories:
  - "其他"
---

天下苦 github 久已，鉴于国内网络环境对 github 的种种限制，导致 github 的使用体验比较差，于是考虑搭建一个私有的 git 代码管理平台。对比了多种方案最后选择了[gogs](https://gogs.io/),主要优点如下：

- 底层基于 go，运行速度很快，资源消耗低
- 功能简介，没有各种花里胡哨的功能
- 代码开源

<!-- more -->

具体部署过程不在这里展开，官网有详细的中文文档。

**PS：不要使用 gogs 自带的 ssh 服务，支持的协议比较少。如果物理机部署直接用物理机的 ssh 就行了，如果 docker 部署可将其它端口映射到 docker 容器的 ssh 端口(22)**

那么问题就产生了，如果有些项目需要推送到 github 怎么办？

配置多个远程地址即可，按需推送到远程。方法如下：

## 支持多个远程地址，实现默认拉取，推送操作私有地址

1. 先将 github 的 remote 名改为 github

```bash
git  remote rename origin github
```

2. 再增加私有平台的 ssh 地址

```bash
git remote add origin ssh://xxxxxxxxxxxxxxxxx.git
```

3. 拉取私有平台的 master 地址

```bash
git pull origin master
```

4. 将本地 master 和私有平台 master 关联起来，实现操作操作私有平台

```bash
git branch --set-upstream-to=origin/master master
```

关联后,执行 git push,git pull 命令默认都是操作的 origin（也就是私有平台）

## 拉取推送 github

操作非默认分支时需要指定远程地址

```bash
# 表示拉取远程 master 分支
git pull github master
# 表示推送到 github 上
git push github
```

## 进行分支切换

当切换到一个远程分支时需要指定远程地址，比如下面命令切换到 origin/dev 分支：

```bash
git checkout --track origin/dev
```

**PS:在进行 push 操作前最后先 pull 一次。**
