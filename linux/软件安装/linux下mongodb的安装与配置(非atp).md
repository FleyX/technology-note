---
id: "2018-11-20-10-38-05"
date: "2018/11/20 10:38:05"
title: "linux下mongodb的配置与安装"
tags: ["mongodb", "linux"]
categories: 
- "linux"
- "软件相关"
---

&emsp;&emsp;首先到官网下载安装包,官网地址如下：[点击跳转](https://www.mongodb.com/download-center/community),选中合适的版本，下面会出现下载链接，然后使用 wget url 下载到当前文件夹下。mongodb 4.04 ubuntu18.04 64 下载命令如下：

```shell
wget https://fastdl.mongodb.org/win32/mongodb-win32-x86_64-2008plus-ssl-4.0.4.zip

```

&emsp;&emsp;然后解压文件到当前文件夹

```shell
tar -zxvf mongodb-linux-x86_64-ubuntu1804-4.0.4.tgz

```

&emsp;&emsp;然后编写配置文件，进入到解压后的目录下，创建文件 mongodb.conf,填入如下内容：

```properties
dbpath = /usr/local/mongodb/data/db    #数据文件存放目录

logpath = /usr/local/mongodb/logs/mongodb.log #日志文件，注意这是文件路径，不是文件夹路径

port = 27017   #端口

fork = true   #以守护程序的方式启用，即在后台运行

auth = true  #开启认证

```

&emsp;&emsp;然后就可以启动

<!-- more -->

```shell
./bin/mongod -f mongodb.conf

```

ps:关于身份认证，可以先将auth设置成false，然后连接mongodb创建用户，创建用户完了后再将auth改成true。