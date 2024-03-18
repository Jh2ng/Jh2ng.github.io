---
title: python imgkit模块
author: jh2ng
cover: 'https://image.jh2ing.com/20230729121448.png'
tags:
  - 学习笔记
categories:
  - Python
abbrlink: 20810
date: 2023-07-27 00:00:00
---

#### 前言
在编写QQ机器人的过程中发现输出文字偏多，排版混乱。通过借鉴其它机器人的解决办法，采用将文字写入图片进行输出，这种方式可以有很多方法实现，比如使用python将文字写入图片的方法，而本文将使用html自定义模板并将html转成图片的方法，灵活度高可自行通过样式去调整文字排版，而且相对比较简单。

#### 安装
安装imgkit模块
```shell
pip3 install imgkit
```
一开始准备在PC端进行测试使用，发现wkhtmltopdf需要安装，于是直接在服务器上测试使用。
> wkhtmltopdf地址：`https://github.com/wkhtmltopdf/wkhtmltopdf`

imgkit需要配置wkhtmltopdf使用，使用centos8进行环境搭建：
```shell
# ubuntu
sudo apt-get install xvfb
# centos
yum install xorg-x11-server-Xvfb

yum install -y fontconfig libXrender libXext xorg-x11-fonts-Type1 xorg-x11-fonts-75dpi
# 图片中文乱码
yum groupinstall Fonts -y
# 下载
wget https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox-0.12.5-1.centos8.x86_64.rpm
# 安装
rpm -ivh wkhtmltox-0.12.5-1.centos8.x86_64.rpm
wkhtmltopdf --version
```

#### 使用
imgkit有三种转图片的方法，分别是`from_url()`、`from_file()`、`from_string()`。
```python
import imgkit

imgkit.from_url('http://baidu.com', 'out.jpg')
imgkit.from_file('template.html', 'out.jpg')
imgkit.from_string('test', 'out.jpg')
```

imgkit有两个配置参数`config`、`options`，即`imgkit.from_string('test', 'out.jpg', config=config, options=options)`
`config`配置wkhtmltoimage和xvfb路径，在windwos下指定wkhtmltoimage.exe:
`config = imgkit.config(wkhtmltoimage='wkhtmltoimag.exe'`。
而options配置的是wkhtmltoimage中的参数，比如指定图片的大小等，注意的是在填写参数时，wkhtmltoimage帮助手册中有`--`需要去掉。
官方示例：
```python
options = {
    'format': 'png',
    'crop-h': '3',
    'crop-w': '3',
    'crop-x': '3',
    'crop-y': '3',
    'encoding': "UTF-8",
    'custom-header' : [
        ('Accept-Encoding', 'gzip')
    ],
    'cookie': [
        ('cookie-name1', 'cookie-value1'),
        ('cookie-name2', 'cookie-value2'),
    ],
    'no-outline': None
}

imgkit.from_url('http://google.com', 'out.png', options=options)
```


获取到所需的数据之后，通过for循环遍历数据将数据填入`<td>`单元格，在body和head部分做好表格样式，然后将填写好的单元格插入进去就可以形成完整的表格数据了。
这里简单的写了一个模板表格，如果需要更加精美的图片可以对css进一步的优化来丰富图片的排版内容。
根据需求写出的部分代码如下：
```python
def tampToTime(timestamp):
    dt = datetime.datetime.fromtimestamp(timestamp)
    formatted_time = dt.strftime('%H:%M:%S')

    return formatted_time

for i in data:
		html = f"""
		<td>{tampToTime(i['createTime'])}</td>
		<td>{i['activity']}</td>
		<td>{i['leader']}</td>
		<td>{i['number']}</td>
		<td>{i['content']}</td>
		"""
		template_html += html
	
	

head = """
<head>
	<style>
		table{
			width: 100%;
			height: 100%;
		}
	</style>
</head>
"""
body = f"""
<body>
	<table border="1" cellspacing="0">
		<thead>
			<tr>
				<th>创建时间</th>
				<th>副本</th>
				<th>团长名</th>
				<th>人数</th>
				<th>备注</th>
			</tr>
		</thead>
		<tbody>
			<tr>
				{template_html}
			</tr>
		</tbody>
	</table>
</body>
		"""
    
result_html = head + body
imgkit.from_string(result_html,"out.jpg")
```
从以上代码看到head和body是分开的，因为css中的`{}`在字符串中代表的是占位符，引入变量格式化成字符串，如`{template_html}`，而样式是`table{width: 100%; height: 100%;}`会把`{}`识别成占位符。
在官方文档中也有更佳有效的方法，官方引入css的案例如下：
```python
# Single CSS file
css = 'example.css'
imgkit.from_file('file.html', options=options, css=css)

# Multiple CSS files
css = ['example.css', 'example2.css']
imgkit.from_file('file.html', options=options, css=css)
```