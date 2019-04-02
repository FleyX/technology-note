---
id: '2019-03-03-15-42'
date: '2019-03-03-15-42'
title: 'docker实现hexo博客自动部署，实时更新'
tags: ['docker', 'hexo', 'next', 'webhook']
categories:
  - '其他'
---

## 一、背景

&emsp;&emsp;你是否有过想要搭建一个 hexo 博客，但是看着那冗长的教程，唉声叹气？
&emsp;&emsp;你是否因为每次发布新的博文，都要重新构建，部署而逐渐放弃写博客？

&emsp;&emsp;现在解决有了完美的解决办法了，一键构建，无需进行复杂的配置，开箱即用。同时支持 github 的 webhook 来实现实时构建，只需就行一次 push 操作，便能自动重新构建发布，无需手动操作。

&emsp;&emsp;详见[hexoBlog 自动构建](https://github.com/FleyX/hexoBlog)

使用方法：

## 从 github 克隆本仓库

```bash
git clone git@github.com:FleyX/hexoBlog.git
```

## 基本配置

1. 修改`docker/docker-compose.yml`文件,指定博文所在 gihub 仓库和 webhook 密钥.

![docker-compose文件修改](https://raw.githubusercontent.com/FleyX/files/master/blog/20190303145035.png)

**github 配置 webhock 步骤如下：**
&emsp;&emsp;以我的博文仓库(technology-note)为例：

- 新增一个 webhook
  ![](https://raw.githubusercontent.com/FleyX/files/master/blog/20190303161438.png)

- 配置 webhook
  ![](https://raw.githubusercontent.com/FleyX/files/master/blog/20190303161713.png)

&emsp;&emsp;将 url 的地址换为你的服务器地址，然后设置 secret 密钥，就 OK 了。再将密钥设置到 docker-compose.yml 中即可。

2. 博文 markdown 文件编写规范,详情参见[分布式事务.md](https://raw.githubusercontent.com/FleyX/technology-note/master/%E6%95%B0%E6%8D%AE%E5%BA%93/%E5%88%86%E5%B8%83%E5%BC%8F/%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1.md)：

<!-- more -->

```yaml
---
id: '2018-10-03-10-58'
date: '2018/10/03 10:58'
title: '分布式事务'
tags: ['分布式', 'sql', '2PC', 'TCC', '异步补偿']
categories:
  - '数据库'
  - '分布式事务'
---

```

参数含义如下：

- id：博文 id，博文链接也会使用这个值
- date: 博文创建日志
- title: 博文标题
- tags: 文章标签
- categories: 文章分类，支持多级分类，第一个最高级依次降低

&emsp;&emsp;如果想实现首页概览，秩序在想要展现的部分下加上`<!-- more -->`,如下所示：

![](https://raw.githubusercontent.com/FleyX/files/master/blog/20190303150138.png)

3. 在 docker 目录下，执行`docker-compose up -d`，完工，访问服务器 IP 或域名即可看到效果。（注意首次部署可能会很慢，取决于网络情况和服务器配置）。

## 详细配置

&emsp;&emsp;上图只是基本配置，下面是常用的配置：

### 设置文章永久链接

&emsp;&emsp;编辑`hexo/_config.yml`下 16，17

![](https://raw.githubusercontent.com/FleyX/files/master/blog/20190303150537.png)

如果部署在根目录下，将 url 设置为`服务器域名`，root 设置为`/`
如果部署在 test 路径下，将 url 设置为`服务器域名/test`,root 设置为`/test`

### 设置站点信息

&emsp;&emsp;编辑`hexo/_config.yml`下 6-10 行，设置博客标题，子标题，关键词，作者等信息

```yaml
title: Hexo
subtitle: To strive, to seek, to find, and not to yield.
description: To strive, to seek, to find, and not to yield.
keywords: ['java', 'node', 'html', 'javascript']
author: fleyX
```

**注意下面的都是配置主题的配置文件,位置`themes/_config.yml`，本博客使用的 Next 主题，其他主题的配置可能不一样**

### 设置社交信息

&emsp;&emsp;编辑第 178 行 social 下项目：

![](https://raw.githubusercontent.com/FleyX/files/master/blog/20190303151851.png)

### 设置打赏

&emsp;&emsp;编辑 327 行 reward 下属性，设置支付宝/微信收款图片，可将图片放到`hexo/source/static/img`目录下。

![](https://raw.githubusercontent.com/FleyX/files/master/blog/20190303152128.png)

### 集成 gitalk 评论

&emsp;&emsp;建议百度如何在 github 中配置 gitalk，这里默认你已经配置完毕，拥有 id 和 secret。编辑 570 行,设置 enable 为 true，然后加入你的信息：

![](https://raw.githubusercontent.com/FleyX/files/master/blog/20190303152638.png)

### 集成 cnzz 统计

&emsp;&emsp;设置 635 行，cnzz id 即可

![](https://raw.githubusercontent.com/FleyX/files/master/blog/20190303152519.png)

其他更加详细配置参看官方文档。
