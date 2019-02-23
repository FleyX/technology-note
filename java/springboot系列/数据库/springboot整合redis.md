---
id: "2019-02-22-14-59"
date: "2019/02/22 14:59:00"
title: "springboot整合redis"
tag: ["java","spring-boot","redis","nosql"]
categories: 
- "java"
- "spring boot学习"
---

&emsp;&emsp;项目源代码在 github，地址为：[https://github.com/FleyX/demo-project/tree/master/mybatis-test](https://github.com/FleyX/demo-project/tree/master/mybatis-test)，有需要的自取。

​&emsp;&emsp;redis作为一个高性能的内存数据库，如果不会用就太落伍了，之前在node.js中用过redis，本篇记录如何将redis集成到spring boot中。提供redis操作类，和注解使用redis两种方式。主要内容如下：

- docker安装redis
- springboot 集成redis
- 编写redis操作类
- 通过注解使用redis

# 安装redis

&emsp;&emsp;通过docker安装，docker compose编排文件如下：
```yml
# docker-compose.yml
version: "3"
services:
  redis:
    container_name: redis
    image: redis:3.2.10
    ports:
      - "6379:6379"
```

&emsp;&emsp;然后在`docker-compose.yml`所在目录使用`docker-compose up -d`命令，启动redis。

# 集成springboot