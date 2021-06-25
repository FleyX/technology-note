---
id: "20210625"
date: "2019/06/25 10:38:05"
title: "linux闪屏问题解决"
tags: ["linux", "arch", "manjaro",“闪屏”]
categories:
  - "linux"
  - "软件安装"
---


本质是因为intel核显的可变刷新率特性问题，在Intel graphics的wiki页面中有描述，[点击跳转查看](https://wiki.archlinux.org/title/intel_graphics#Screen_flickering)

解决办法是在内核参数中禁用此特性,方法如下：

1. grub启动页面增加（临时解决，重启失效）

按e编辑启动参数，在`kernel`开头的那一行最后增加`i915.enable_psr=0`

2. 编辑`/etc/default/grub`文件（永久解决)

在`GRUB_CMDLINE_LINUX_DEFAULT`开头的行最后增加`i915.enable_psr=0`