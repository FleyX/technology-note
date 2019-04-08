---
id: '2019-02-28-15-50'
date: '2019/02/28 15:50'
title: 'java-jwt生成与校验'
tags: ['java', 'jwt', 'java-jwt', 'token']
categories:
  - 'java'
  - 'java工具集'
---

**本篇原创发布于：**[FleyX 的个人博客](http://www.tapme.top/blog/detail/2019-02-28-15-50)

# 什么是 JWT

&emsp;&emsp;这里是[jwt 官方地址](https://jwt.io/),想了解更多的可以在这里查看。

&emsp;&emsp;jwt 全称是**JSON Web Token**,从全称就可以看出 jwt 多用于认证方面的。这个东西定义了一种简洁的，自包含的，安全的方法用于通信双方以 json 对象的形式传递信息。其中**简洁**，**安全**，**传递信息**和 web 系统非常契合。

&emsp;&emsp;jwt 实际上就是一个字符串，由以下三个部分构成(通过.分隔)：

- Header 头部
- Payload 负载
- Signature 签名

因此一个 jwt 字符串都是如下的形式：
Header.Payload.Signature

## Header

&emsp;&emsp;header 大多数情况下是只包含两个属性的 json 字符串,token 的类型("JWT")和用到的算法(比如 HS256,RS256,ES256 等)如下：

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

然后用 Base64 将其编码就等到了 jwt 的第一部分

## Payload

&emsp;&emsp;payload 顾名思义用于携带数据的，这里的数据有三种类型：

- Registered claims：一组预定义的声明，写在 jwt 标准中，所有对其的实现都要准守。但不是强制要求携带。有以下几个字段：iss(签发者),iat(创建时间),exp(过期时间),aud(签发者),sub(面向的用户)
- public claims：随意定义，通常存放用户 id，用户类别等非铭感信息

这些数据也是 json 的形式，用 Base64 编码后就得到了 JWT 的第二个部分。

## Signature

&emsp;&emsp;签名就是通过设定的秘钥和签名算法来对 header 和 payload 进行签名得到一个签名字符串，将这三个字符串组合起来就是 JWT 了。

# java 中使用

&emsp;&emsp;通过`java-jwt`来实现，首先引入依赖：

```xml
<dependency>
    <!-- 截止当前最新版本为3.7 -->
    <groupId>com.auth0</groupId>
    <artifactId>java-jwt</artifactId>
    <version>3.7.0</version>
</dependency>
```

## 签名

&emsp;&emsp;使用 HMC256，代码入下：

```java
    private static final Algorithm ALGORITHM= Algorithm.HMAC256("security");
    public static String encode() {
        //通过秘钥生成一个算法
        String token = JWT.create()
                //设置签发者
                .withIssuer("test")
                //设置过期时间为一个小时
                .withExpiresAt(new Date(System.currentTimeMillis()+60*60*1000))
                //设置用户信息
                .withClaim("name","小明")
                .withClaim("age",20)
                .sign(ALGORITHM);
        return token;
    }
```

## 验证

&emsp;&emsp;验证代码如下：

```java

    //校验类
    private static final JWTVerifier JWT_VERIFIER= JWT.require(ALGORITHM).withIssuer("test").build();
    public static void decode(String token) {
        DecodedJWT decodedJWT = JWT_VERIFIER.verify(token);
        //如果校验失败会抛出异常
        //payload可从decodeJWT中获取
    }
```
