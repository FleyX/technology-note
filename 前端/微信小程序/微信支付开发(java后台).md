---
id: "20220727"
date: "2022/07/27 21:05"
title: "手把手教你springboot集成微信支付"
excerpt: "最近要做一个微信小程序，需要微信支付，所以研究了下怎么在java上集成微信支付功能，特此记录"
index_img: https://qiniupic.fleyx.com/blog/202207271552955.png?imageView2/2/w/200
banner_img: https://qiniupic.fleyx.com/blog/202207271552955.png
tags: ["java", "spring-boot", "微信支付", "小程序"]
categories:
  - "java"
  - "微信支付"
---

最近要做一个微信小程序，需要微信支付，所以研究了下怎么在 java 上集成微信支付功能，特此记录下。

**本文完整代码:[点击跳转](https://git.fleyx.com/fanxb/demo-project/src/master/weChatPay)**

## 准备工作

### 小程序开通微信支付

1. 首先需要在微信支付的官网[点击跳转](https://pay.weixin.qq.com/index.php/core/home)上注册一个服务商
2. 在服务商的管理页面中申请关联小程序,通过小程序的 appid 进行关联
3. 进入微信公众平台，功能-微信支付中确认关联（如果服务商和小程序的注册主体不一样，还要经过微信的审核）

### 获取各种证书、密钥文件

这里比较麻烦，需要认真点。

目前微信支付的 api 有 V2 和 V3 两个版本，V2 是 xml 的数据结构不建议用了，很麻烦（虽然 V3 也不简单）.

**以下内容全部基于微信支付 V3 的版本**

你需要获取如下东西：

- 商户 id：这个可以在小程序微信公众平台-功能-微信支付 页面中的已关联商户号中得到
- 商户密钥：这个需要在微信支付的管理后台中申请获取
- 证书编号: 同样在微信支付的管理后台中申请证书，申请证书后就会看到证书编号
- 证书私钥：上一步申请证书的时候同时也会获取到证书的公钥、私钥文件。开发中只需要私钥文件

## 代码开发

由于支付属于比较敏感的操作，所以建议将参数配置放在后台，前端请求获取到参数后直接调起微信支付。

整个微信支付流程如下：

1. 小程序端请求后台获取统一支付参数
2. 后台调用微信 api([官方文档](https://pay.weixin.qq.com/wiki/doc/apiv3/apis/chapter3_5_1.shtml))生成预订单，并构造统一下单接口的参数返回小程序
3. 小程序根据参数调用统一下单接口([官方文档](https://pay.weixin.qq.com/wiki/doc/apiv3/apis/chapter3_5_4.shtml))

实际开发中，小程序端的开发内容很少，98%的工作量都在后台。

## 准备请求 htpClient

虽然微信官方目前没有正式放出官方 sdk，但是 github 上已经有了 sdk 的仓库，目前正在开发中，[地址](https://github.com/wechatpay-apiv3/wechatpay-apache-httpclient).这个工具做了一定程度的封装，极大简化了加密解密证书配置等繁琐的操作。

和微信的请求需要做双向加密，因此要在系统启动时创建一个专用的 httpClient，用来调用微信支付 api.代码如下：

```java
@PostConstruct
public void init() throws Exception {
    log.info("私钥路径:{}", certKeyPath);
    PrivateKey merchantPrivateKey = PemUtil.loadPrivateKey(new FileInputStream(certKeyPath));
    // 获取证书管理器实例
    certificatesManager = CertificatesManager.getInstance();
    sign = SecureUtil.sign(SignAlgorithm.SHA256withRSA, merchantPrivateKey.getEncoded(), null);

    // 向证书管理器增加需要自动更新平台证书的商户信息
    certificatesManager.putMerchant(merchantId, new WechatPay2Credentials(merchantId,
            new PrivateKeySigner(merchantSerialNumber, merchantPrivateKey)), apiV3Key.getBytes(StandardCharsets.UTF_8));
    // 从证书管理器中获取verifier
    Verifier verifier = certificatesManager.getVerifier(merchantId);
    WechatPayHttpClientBuilder builder = WechatPayHttpClientBuilder.create()
            .withMerchant(merchantId, merchantSerialNumber, merchantPrivateKey)
            .withValidator(new WechatPay2Validator(verifier));
    httpClient = builder.build();
}
```

## 预定单生成并返回小程序请求参数

代码如下：

```java
JSONObject obj = new JSONObject();
        obj.put("mchid", merchantId);
        obj.put("appid", appId);
        obj.put("description", body.getDescription());
        obj.put("out_trade_no", body.getSn());
        obj.put("notify_url", "https://backend/asdfasdf/notify");
        Map<String, String> attach = new HashMap<>();
        attach.put("sn", body.getSn());
        obj.put("attach", JSON.toJSONString(attach));
        JSONObject amount = new JSONObject();
        amount.put("total", body.getPrice().multiply(BigDecimal.valueOf(100)).intValue());
        obj.put("amount", amount);
        JSONObject payer = new JSONObject();
        //放入用户的openId
        payer.put("openid", "");
        obj.put("payer", payer);
        log.info("请求参数为：" + JSON.toJSONString(obj));
        HttpPost httpPost = new HttpPost("https://api.mch.weixin.qq.com/v3/pay/transactions/jsapi");
        httpPost.addHeader("Accept", "application/json");
        httpPost.addHeader("Content-type", "application/json; charset=utf-8");
        httpPost.setEntity(new StringEntity(obj.toJSONString(), "UTF-8"));
        try {
            CloseableHttpResponse response = httpClient.execute(httpPost);
            String bodyAsString = EntityUtils.toString(response.getEntity());
            String prePayId = JSONObject.parseObject(bodyAsString).getString("prepay_id");

            //准备小程序端的请求参数
            Map<String, String> map = new HashMap<>(6);
            map.put("appId", appId);
            String timeStamp = String.valueOf(now / 1000);
            map.put("timeStamp", timeStamp);
            String nonceStr = IdUtil.fastSimpleUUID();
            map.put("nonceStr", nonceStr);
            String packageStr = "prepay_id=" + prePayId;
            map.put("package", packageStr);
            map.put("signType", "RSA");
            // 注意\n必须添加
            map.put("paySign", Base64.encode(sign.sign(appId + "\n" + timeStamp + "\n" + nonceStr + "\n" + packageStr + "\n")));
            return map;
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
```

**注意在生成小程序请求参数 paySign 时，\n 必须添加**

## 小程序端请求

小程序端获取请求参数后，直接调用`wx.requestPayment(后台返回的参数)`,即可调起支付

## 支付成功回调

微信支付成功后，会通知服务端支付成功，通过之前配置的回调接口。注意回调接口必须为 https。

微信回调参数也是加密的，必须要经过解密后才能获取，代码如下：

**注意：部分参数是通过请求头提供的，nginx 等代理在转发请求时可能会将请求头过滤掉，导致无法获取对应参数**

```java
@Override
    public void notify(HttpServletRequest servletRequest) {

        String timeStamp = servletRequest.getHeader("Wechatpay-Timestamp");
        String nonce = servletRequest.getHeader("Wechatpay-Nonce");
        String signature = servletRequest.getHeader("Wechatpay-Signature");
        String certSn = servletRequest.getHeader("Wechatpay-Serial");

        try (BufferedReader reader = new BufferedReader(new InputStreamReader(servletRequest.getInputStream()))) {
            StringBuilder stringBuilder = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                stringBuilder.append(line);
            }
            String obj = stringBuilder.toString();
            log.info("回调数据:{},{},{},{},{}", obj, timeStamp, nonce, signature, certSn);
            Verifier verifier = certificatesManager.getVerifier(merchantId);
            String sn = verifier.getValidCertificate().getSerialNumber().toString(16).toUpperCase(Locale.ROOT);
            NotificationRequest request = new NotificationRequest.Builder().withSerialNumber(sn)
                    .withNonce(nonce)
                    .withTimestamp(timeStamp)
                    .withSignature(signature)
                    .withBody(obj)
                    .build();
            NotificationHandler handler = new NotificationHandler(verifier, apiV3Key.getBytes(StandardCharsets.UTF_8));
            // 验签和解析请求体
            Notification notification = handler.parse(request);
            JSONObject res = JSON.parseObject(notification.getDecryptData());
            //做一些操作
            JSONObject attach = res.getJSONObject("attach");
        } catch (Exception e) {
            log.error("微信支付回调失败", e);
        }
    }
```

## 拓展

以上只有关键的支付和回调接口示例，其他的接口请求也是类似的，就不一一列出。

---
