---
title: brup免java版本切换解决
cover: 'https://image.jh2ing.com/20230624131622.png'
tags:
  - 环境搭建
categories:
  - 网络安全
abbrlink: 62204
date: 2023-06-24 00:00:00
---

burp suite新版需要较高的jdk版本才能运行，而其他工具或开发环境一般默认使用的是jdk8，所以burp suite经常会遇到java版本切换的问题，网上也有许多的解决方案。比较常用的是使用jdk切换工具。
最近发现一种解决办法，原理如同`python2`和`python3`,配置多个环境变量，然后修改`bin`目录下的`java.exe`,例如把`java.exe`改为`java8.exe`，这样就可以使用`java8`来运行脚本。其他就使用默认的系统`java`的版本。但是这种方法运行`BurpLoaderKeygen.jar`运行时无法找到jdk，必须将高版本的jdk设置成默认的jdk才可以（也就是必须修改jdk8的java.exe）。然后使用jdk8就使用`java8 -jar`来运行jar包。由于许多东西都需要使用jdk8，所以这种方法就pass了。
又选择了写一个burp suite的启动脚本来免java版本切换；这里记录一下过程。
1、先去镜像站下载jdk，这里下载了jdk-18.0.2.1。
2、在`jdk-18.0.2.1\bin`下运行`java.exe -jar BurpLoaderKeygen.jar`。
3、然后激活burp。
> 注意运行的是当前目录下的java.exe而不是系统环境变量中的java.exe，可以先version查看一下。

4、激活成功后在`jdk-18.0.2.1\bin`中编写`burp.bat`，内容如下：
```bat
@echo off
START ./javaw --add-opens=java.desktop/javax.swing=ALL-UNNAMED --add-opens=java.base/java.lang=ALL-UNNAMED --add-opens=java.base/jdk.internal.org.objectweb.asm=ALL-UNNAMED --add-opens=java.base/jdk.internal.org.objectweb.asm.tree=ALL-UNNAMED --add-opens=java.base/jdk.internal.org.objectweb.asm.Opcodes=ALL-UNNAMED -javaagent: ../burp/BurpLoaderKeygen.jar -noverify -jar..\burp\burpsuite_pro_v2023.6.jar
```
> 注意：这里路径做了处理，根据实际路径修改。这段参数复制自`BurpLoaderKeygen.jar`中的`Loader Command`。只需要后面的参数部分。使用javaw来运行，不会弹出命令行窗口。


然后创建桌面快捷方式，为了好看也可以修改一下图标。
这里提供我下载好的图标，可以直接使用。
![burp_suite_alt_macos_bigsur_icon_190318](https://image.jh2ing.com/burp_suite_alt_macos_bigsur_icon_190318.ico)

网上也有类似的免环境配置的brup提供下载，为了安全起见，还是自己配置一下吧。
效果图：
![20230624131622](https://image.jh2ing.com/20230624131622.png)