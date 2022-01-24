---
id: "20220121"
date: "2022/01/21 10:38:05"
title: "免费ssl证书看这篇就够了(单域名/dnspod泛域名申请/自动续期)"
tags: ["linux", "nginx", "ssl", "certbot", "泛域名", "dnspod"]
index_img: https://qiniupic.fleyx.com/blog/202201241640756.png
banner_img: https://qiniupic.fleyx.com/blog/202201241641133.jpg
categories:
  - "linux"
  - "软件使用"
---

不知道你发现了没有，如果使用 chrome 浏览器访问某些网站，在网址左边会有如下的安全提示：

![不安全](https://qiniupic.fleyx.com/blog/202201212029672.png)

原因就是因为此网站协议为 http，未进行 ssl 加密。目前全球网站都已进入 https 时代，因此非常有比较将自己的站点升级为`https`.


<!-- more -->

个人使用没必要付费购买 ssl 证书，目前有不少免费 ssl 证书申请的网站，比如：

- Let's Encrypt
- TrustAsia
- DigiCert

其中只有`Let's Encrypt`限制最少，能够申请主域名，子域名，泛域名证书，因此本文以`Let's Encrypt`为例讲解如何进行证书申请。

## 准备工作

首先需要安装 certbot 软件，用于进行证书申请。这里以 debian 系列(ubuntu 之类也可用),nginx 为例,其他系统可在官网查看安装步骤：[点击跳转](https://certbot.eff.org/instructions?ws=nginx&os=debianstretch),选择 web 软件和系统即可：

![选项](https://qiniupic.fleyx.com/blog/202201241015938.png)

1. 安装`snapd`

```bash
sudo snap install core; sudo snap refresh core
```

2. 安装`certbot`

```bash
sudo snap install --classic certbot
```

3. 添加路径到 PATH

使用`echo $PATH`命令，如果路径中存在`/snap/bin`护理下面操作。

为了方便使用，将 snapd 可执行文件路径添加到 PATH 中，编辑`/etc/profile`文件,在最后一行增加`export PATH=$PATH:/snap/bin`

## nginx 单域名申请 ssl 证书

仅生成证书不修改配置,如果想要 certbot 直接将 ssl 证书配置到 nginx 中，可以去掉`certonly`选项.-d 后更要生成证书的域名

```bash
sudo certbot certonly --nginx -d a.test.com -d b.test.com
```

此命令会自动配置证书更新，无需手动重新申请。可通过如下命令验证：

```bash
sudo certbot renew --dry-run
```

## 泛域名申请

泛域名证书申请较为麻烦，不能像单域名一样使用 web 服务器验证，一般通过配置 dns 来进行验证，因此就需要自动化的设置 dns 配置来进行自动续签证书。

目前 certbot 官方支持的 dns 操作工具中没有`dnspod`,支持列表如下：

- certbot-dns-cloudflare
- certbot-dns-cloudxns
- certbot-dns-digitalocean
- certbot-dns-dnsimple
- certbot-dns-dnsmadeeasy
- certbot-dns-gehirn
- certbot-dns-google
- certbot-dns-linode
- certbot-dns-luadns
- certbot-dns-nsone
- certbot-dns-ovh
- certbot-dns-rfc2136
- certbot-dns-route53
- certbot-dns-sakuracloud

好在万能的 github 总是有解决办法的，我这里使用的这个库：[certbot-auth-dnspod](https://github.com/al-one/certbot-auth-dnspod)

步骤如下：

1. 密钥申请

首先需要申请 dnspod 的密钥，在个人中心->API 密钥界面可申请。申请完毕后会得到一个 id 和一个 token。

2. 配置到文件中

将id和token用","分隔放到`/etc/dnspod_token`

3. 下载脚本

```bash
$ wget https://raw.githubusercontent.com/al-one/certbot-auth-dnspod/master/certbot-auth-dnspod.sh
$ chmod +x certbot-auth-dnspod.sh
```

4. 生成ssl证书

使用如下命令进行ssl证书生成：

```bash
sudo certbot certonly --manual --preferred-challenges dns-01 --email test@test.com -d laravel.run -d *.laravel.run --server https://acme-v02.api.letsencrypt.org/directory --manual-auth-hook /path/to/certbot-auth-dnspod.sh --manual-cleanup-hook "/path/to/certbot-auth-dnspod.sh clean"
```

自定义的参数如下：

- --email  后接邮箱
- -d 后接要生成证书的地址

**注意要将脚本路径修改为你下载文件的真实路径**


