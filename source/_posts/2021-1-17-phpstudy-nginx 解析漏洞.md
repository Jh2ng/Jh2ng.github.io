---
title: phpstudy_nginx 解析漏洞
author: jh2ng
cover: 'https://b3logfile.com/file/2020/11/image-d2a9dfb3.png'
tags:
  - vulhub
categories:
  - 漏洞复现
abbrlink: 19714
date: 2021-01-17 16:13:00
---

## 漏洞简介

2020年09月03日，360CERT监测发现 phpstudy 发布了 phpstudy 安全配置错误漏洞 的风险通告，漏洞等级：高危，漏洞评分：7.6。

phpStudy 存在 nginx 解析漏洞，攻击者能够利用上传功能，将包含恶意代码的合法文件类型上传至服务器，从而造成任意代码执行的影响。

## 影响版本

phpstudy:phpstudy : <=8.1.0.7

## 环境搭建

主要是nginx配置不当造成的解析漏洞，与php、nginx版本无关，但在高版本的php中，由于“security.limit_extensions”的引入，使得该漏洞难以被成功利用。

环境配置如下（win10 + phpstudy）

![image.png](https://b3logfile.com/file/2020/11/image-292148ed.png)

写一个文件上传页面上传文件(html+php)

![image.png](https://b3logfile.com/file/2020/11/image-7d89efa0.png)

## 漏洞复现

上传图片马，`copy 1.jpg/b + tset.php/a webshell.jpg `这个命令制作完成后有点问题，所以直接更改php的后缀，也可以在网上找图片马。

![image.png](https://b3logfile.com/file/2020/11/image-f492fef6.png)

![image.png](https://b3logfile.com/file/2020/11/image-37ac7baf.png)

这里得到了图片存储的路径，然后访问这张图片 `http://192.168.73.146/upload/test.jpg`

![image.png](https://b3logfile.com/file/2020/11/image-be3ca030.png)

直接使用paylaod `http://192.168.73.146/upload/test.jpg/xxx.php`造成解析漏洞

![image.png](https://b3logfile.com/file/2020/11/image-d2a9dfb3.png)

## 漏洞分析

`http://192.168.73.146/upload/test.jpg/xxx.php ` 访问一个不存在的php文件为什么对上一层的jpg文件进行了解析？既然是配置不当，就去看看它们的配置文件。

```
server {
        listen        80;
        server_name  localhost;
        root   "C:/phpstudy_pro/WWW";
        location / {
            index index.php index.html;
            error_page 400 /error/400.html;
            error_page 403 /error/403.html;
            error_page 404 /error/404.html;
            error_page 500 /error/500.html;
            error_page 501 /error/501.html;
            error_page 502 /error/502.html;
            error_page 503 /error/503.html;
            error_page 504 /error/504.html;
            error_page 505 /error/505.html;
            error_page 506 /error/506.html;
            error_page 507 /error/507.html;
            error_page 509 /error/509.html;
            error_page 510 /error/510.html;
            autoindex  off;
        }
        location ~ \.php(.*)$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_split_path_info  ^((?U).+\.php)(/?.+)$;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_param  PATH_INFO  $fastcgi_path_info;
            fastcgi_param  PATH_TRANSLATED  $document_root$fastcgi_path_info;
            include        fastcgi_params;
        }
}
```

通过正则找到php文件并把它交给fastcgi处理,`fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;` 这个配置的意思是在浏览器中访问.php文件，读取$document_root（网站根目录）下的.php文件。

```
访问此URL http://192.168.73.146/upload/test.jpg/xxx.php 
nginx 传递给fastcgi的值 /../../upload/test.jpg/xxx.php
```

这里怎么访问了上层的test.jpg文件呢？在fastcgi处理xxx.php这个文件的时候，发现没有这个php文件，这时 `php.ini` 配置文件中 `cgi.fix_pathinfo=1` 这个参数就会修复路径，采用上层的路径

```
假设一个路径,cgi依次检查
/../../upload/test.jpg/xxx.php/aaa.php/bbb.php
/../../upload/test.jpg/xxx.php/aaa.php
/../../upload/test.jpg/xxx.php
/../../upload/test.jpg
```

到了这里 `fastcgi_param  SCRIPT_FILENAME` 就成了 `/../../upload/test.jpg` 

![image.png](https://b3logfile.com/file/2020/11/image-37f7410c.png)

所以访问 `http://192.168.73.146/upload/test.jpg/xxx.php` 实际上变成了 `http://192.168.73.146/upload/test.jpg` 并当成了php文件执行。

在是php-fpm.conf中的 `security.limit_extensions`配置了fastcgi解析文件的类型，如果默认为空那么是什么类型的文件就执行什么类型的文件。

## 漏洞修复

1. 将php.ini文件中的cgi.fix_pathinfo的值设置为0，这样它就不会修复路径，你访问xxx.php这个不存在的文件时就会显示此文件不存在。
2. php-fpm.conf中的security.limit_extensions后面的值设置为.php。

## 参考链接

> [https://blog.csdn.net/ll641058431/article/details/53350305]()
>
> [https://www.laruence.com/2010/05/20/1495.html](https://www.laruence.com/2010/05/20/1495.html)
>
> [https://www.cnblogs.com/yuzly/p/11208742.html](https://www.cnblogs.com/yuzly/p/11208742.html)
