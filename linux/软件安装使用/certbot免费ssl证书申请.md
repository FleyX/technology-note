---
id: "20220121"
date: "2022/01/21 10:38:05"
title: "免费ssl证书看这篇就够了(单域名/泛域名/自动续期)"
tags: ["linux", "nginx", "ssl", "certbot", "泛域名"]
hid: true
index_img:
banner_img: 
categories:
  - "linux"
  - "软件使用"
---

不知道你发现了没有，如果使用chrome浏览器访问某些网站，在网址左边会有如下的安全提示：

![不安全](https://qiniupic.fleyx.com/blog/202201212029672.png)

原因就是因为此网站协议为http，未进行ssl加密。目前全球网站都已进入https时代，因此非常有比较将自己的站点升级为`https`.

个人使用没必要付费购买ssl证书，目前有不少免费ssl证书

