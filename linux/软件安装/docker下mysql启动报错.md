---
id: "2018-12-26-11-50"
title: "docker下mysql启动报错"
headWord: "报错是这么产生的，使用装有mysql的镜像创业一个容器，然后在容器中启动mysql就会报错，启动失败。"
tags: ["docker","mysql"]
categories: 
- "linux"
- "踩坑"
---

# 1、报错过程

&emsp;&emsp;报错是这么产生的，使用装有 mysql 的镜像创业一个容器，然后在容器中启动 mysql 就会报错，启动失败。报错内容如下：

```log
2017-11-15T06:44:22.141481Z 0 [ERROR] Fatal error: Can't open and lock privilege tables: Table storage engine for 'user' doesn't have this option
```

# 2、怎么解决

&emsp;&emsp;最开始看到这个报错是比较莫名其妙的，不知道如何解决，百度上搜索资料也不多，找了半天才在`stack overflow`上找到了原因和解决办法，由于 docker 默认的存储驱动是 overlayfs（overlay2),将其改为 aufs 即可，编辑/etc/docker/daemon.json（如果没有这个文件，新建）

```json
{
  "storage-driver": "aufs",
  "debug": true,
  "experimental": true
}
```

关于这个问题，github 上有反馈这个问题，详情[看这里](https://github.com/moby/moby/issues/35503)
