---
id: '2019-03-01-18-52'
date: '2019/03/01 18:52'
title: 'spring boot 基于JWT实现单点登录'
tags: ['spring-boot', 'SSO', '单点登录', 'jwt']
categories:
  - 'java'
  - 'spring boot学习'
---

**本篇原创发布于：**[FleyX 的个人博客](http://tapme.top/blog/detail/2019-03-01-18-52)

照例配个图：
![塞尔达，林克](https://raw.githubusercontent.com/FleyX/files/master/blogImg/20190301190816.png)

<div style="text-align:center">看我大塞尔达，不！是林克</div>

&emsp;&emsp;最近我们组要给负责的一个管理系统 A 集成另外一个系统 B，为了让用户使用更加便捷，避免多个系统重复登录，希望能够达到这样的效果——用户只需登录一次就能够在这两个系统中进行操作。很明显这就是**单点登录(Single Sign-On)**达到的效果,正好可以明目张胆的学一波单点登录知识。

本篇主要内容如下：

- SSO 介绍
- SSO 的几种实现方式对比
- 基于 JWT 的 spring boot 单点登录实现

**注意:**
&emsp;&emsp;SSO这个概念已经出现很久很久了，目前各种平台都有非常成熟的实现，比如`OpenSSO`，`OpenAM`，`Kerberos`，`CAS`等,当然很多时候成熟意味着复杂。本文不讨论那些成熟方案的使用,也不考虑SSO在CS应用中的使用。

# 什么是SSO

&emsp;&emsp;