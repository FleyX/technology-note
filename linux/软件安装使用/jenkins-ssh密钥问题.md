---
id: '20211028'
date: '2021-10-28 15:42:00'
title: 'jenkins配置ssh密钥相关问题'
tags: ['linux', 'ssh-rsa']
categories:
  - 'linux'
  - 'program'
---

最近在使用 jenkins 时，发现了一个比较麻烦的问题，配置 ssh 密钥后，使用密钥登陆远程主机会报错。主要有两种问题：

## jenkens 报错`"C:\\Users\\JE~1\\AppData\\Local\\Temp\\ssh2142299850576289882.key": invalid format`

类似上面 jenkins 日志打印的错误，说无效的格式,出现这个问题的根本原因是 jenkins 支持的密钥格式比较旧，

<!-- more -->

```
-----BEGIN OPENSSH PRIVATE KEY-----
```

查看自己 ssh 密钥如果开始是上面的文本，说明是 jenkins 不支持的格式，配置到 jenkins 中会报错`invalid format `

### 解决办法

指定使用旧的格式即可，如下：

```bash
ssh-keygen -m PEM -t rsa -m "test"
```

增加`-m PEM`参数以使用旧的参数

## 远程主机报错`pubkey: key type ssh-rsa not in PubkeyAcceptedAlgorithms`

这个错误原因是因为最近 open ssh 的新版本中，已经废弃了对 ssh-rsa 的支持。

### 解决办法

编辑**远程主机**的 ssh 服务端配置文件，`/etc/ssh/sshd_config`,增加如下配置:

```conf
PubkeyAcceptedKeyTypes=+ssh-rsa
```

然后重启 ssh 就行了