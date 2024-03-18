---
title: Apache shiro系列漏洞
author: jh2ng
cover: 'https://image.jh2ing.com/image-20230312154511213.png'
tags:
  - vulhub
categories:
  - 漏洞复现
abbrlink: 18014
date: 2023-03-12 17:10:00
---

### Apache shiro 1.2.4反序列化漏洞(CVE-2016-4437)

#### 漏洞描述

Apache Shiro是一款开源安全框架，提供身份验证、授权、密码学和会话管理。Shiro框架直观、易用，同时也能提供健壮的安全性。

Apache Shiro 1.2.4及以前版本中，加密的用户信息序列化后存储在名为remember-me的Cookie中。攻击者可以使用Shiro的默认密钥伪造用户Cookie，触发Java反序列化漏洞，进而在目标机器上执行任意命令。

#### 影响版本

`Apache Shiro <= 1.2.4`

#### 漏洞复现

>在 1.2.4 版本前，默认ASE秘钥Key: `kPH+bIxk5D2deZiIxcaaaA==`
>
>工具：`https://github.com/SummerSec/ShiroAttack2`

登录时查看返回包是否含有`rememberMe`字段，如果没有则将请求包的cookie字段设置为`Cookie: rememberme=1`。

![image-20230312163236271](https://image.jh2ing.com/image-20230312163236271.png)




#### 修复建议

- 升级版本
- 修改密钥

### Apache Shiro 认证绕过漏洞（CVE-2020-1957）

#### 漏洞描述

Apache Shiro是一款开源安全框架，提供身份验证、授权、密码学和会话管理。Shiro框架直观、易用，同时也能提供健壮的安全性。

在Apache Shiro 1.5.2以前的版本中，在使用Spring动态控制器时，攻击者通过构造`..;`这样的跳转，可以绕过Shiro中对目录的权限限制。

#### 影响版本

`Apache Shiro < 1.5.2 `

#### 漏洞复现

直接请求管理页面`/admin/`，会被重定向到登录页面。

构造恶意请求`/xxx/..;/admin/`，即可绕过权限校验，访问到管理页面：

![image-20230312210005083](https://image.jh2ing.com/image-20230312210005083.png)

#### 修复建议

- 升级版本
