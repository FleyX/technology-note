---
id: "20190516"
date: "2019/05/16 10:38:05"
title: "Linux下使用nextcloud搭建个人网盘"
tags: ["linux", "nextcloud", "个人网盘"]
categories:
  - "linux"
  - "软件安装"
---

市面上有那么多的网盘服务提供商，为什么还要自己搭建网盘呢？主要有以下原因:

- 免费的网盘都有种种限制，要么不限速容量小(onedriver,google driver)，要么容量大限速（百度云）
- 付费网盘服务又太贵，穷逼用不起
- 数据放在别人的服务器不安全，说不定就变成 8s 了
- 瞎折腾有趣

两三个月前，矿难无情人友情，三百块入手了一台 4 盘位的 nas 主机，装上 ubuntu，就开始了折腾。

![主界面](https://raw.githubusercontent.com/FleyX/files/master/blogImg/20190516134611.png)

为什么要选择 nextcloud 呢？

- 开源
- 各个平台都有客户端，方便管理
- 功能很完善

<!-- more -->

下面开始正文，搭建 nextcloud。

推荐使用 docker 来搭建环境，非常方便。

1. 首先安装 docker 环境，参考这篇:[docker 安装](https://blog.fleyx.com/blog/detail/1.linux下mongodb的配置与安装)

2. 安装 docker-compose

```bash
sudo apt-get install docker-compose
```

3. 编写 docker-compose.yml

```yaml
version: "2"
services:
  nextcloud:
    image: nextcloud
    container_name: nextcloud
    volumes:
      - /home/nextcloud:/var/www/html
    ports:
      - 8080:80
```

4. 启动

docker-compose.yml 文件所在目录执行`docker-compose up -d`。便能够通过访问 ip+端口,进入 web 端界面。在设置界面可以调成中文。默认进入是英文。

~~提供一个测试账号：ali.tapme.top:8007 test/testgggg~~(已失效)

请勿恶意大量上传下载哦！

