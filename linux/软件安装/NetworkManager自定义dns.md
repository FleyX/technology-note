---
id: "20211019"
date: "2021-10-19 15:42:00"
title: "NetworkManager如何自定义dns,永久生效"
tags: ["linux", "NetworkManger"]
categories:
  - "linux"
  - "program"
---
## 前言

目前比较新的linux发行版都默认使用NetworkManger来管理网络了，然后如何自定义dns就成了一个比较麻烦的事，每次NetworkManager启动都会覆盖`/etc/resolv.conf`文件，特别是使用dhcp获取ip时，dns地址会变成dhcp服务器默认的dns.那么有哪些解决办法呢？

## 修改/etc/resolv.conf

既然这个文件会被NetworkManager修改，那么让它改不了就行了。将resolv.conf设置为不可修改。命令如下：
```bash
sudo chattr +i /etc/resolv.conf
```
这样我们自定义dns后就不会被NetworkManager重新覆盖了。

<!-- more -->

## 更幽雅的配置

上面虽然能达到目的，但是不太幽雅。其实NetworkManager是支持自定义dns的，办法如下:

1. 修改/etc/NetworkManager/conf.d/dns.conf(如没有此文件，新建即可),增加如下两行配置：

```conf
[main]
dns=null
```

2. 修改/etc/NetworkManager/conf.d/dns-servers.conf(如没有此文件，新建即可),增加如下两行配置,设置自定义dns：
```conf
[global-dns-domain-*]
servers=::1,127.0.0.1,8.8.8.8
```

3. 重启软件`sudo systemctl restart NetworkManager`

大功告成～

