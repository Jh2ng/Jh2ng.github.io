---
title: tomcat配置及Servlet入门
author: jh2ng
cover: 'https://img.hacpai.com/file/2019/11/Servlet执行原理-5d47a4a9.PNG'
tags: Java Web
categories:
  - Java
abbrlink: 64929
date: 2019-11-24 16:40:00
---

### Web基础

web资源分为静态和动态资源，静态资源所有用户访问得到的结果都是一摸一样的，静态资源可以直接被浏览器解析。动态资源每个用户访问相同资源后，得到的结果可能不一样，动态资源访问之后，先转换成静态资源，再返回给浏览器。

>什么是静动分离？
>动静分离是让动态网站里的动态网页根据一定规则把不变的资源和经常变的资源区分开来，动静资源做好了拆分以后，我们就可以根据静态资源的特点将其做缓存操作，这就是网站静态化处理的核心思路

>为什么要静动分离？
>当我们去访问网站的时候，有些请求是后台处理，而有些请求则不需要后台处理，为了减少资源的访问次数，所以采用静动分离，提高网站吞吐量。

通信三要素

* IP
* 端口
* 协议

常见的java服务器软件

* webLogic
* webSphere
* JBOSS
* Tomcat

###  tomcat 配置

下载地址`http://tomcat.apache.org/`
安装:直接解压即可 （注意：安装目录不要有中文和空格）

目录结构
![tomcat目录结构.png](https://img.hacpai.com/file/2019/11/tomcat目录结构-83f00ee6.png)

启动 文件目录下的bin目录中的`startup.bat`文件双击运行 ，Linux有`startup.sh`文件
出现的问题主要有运行窗口一闪而过（原因：java环境配置问题。正确配置JAVA_HOME环境）和启动报错（端口占用问题。）

部署项目的方式
1、直接放在webapps目录下或者将项目打包成war包，在将包放在webapps目录下（war自动解压）
2、配置conf/server.xml文件，在<Host>标签体中配置`<Context docBase="项目存放的路径" path="访问路径
" />`
3、在conf\Catalina\localhost创建任意名称的xml文件。在文件中编写`<Context docBase="项目存放的路径" />` 访问的虚拟目录就是XML文件的文件名

### Servlet 入门

#### 实现Servlet接口

```java
public class ServletDemo01 implements Servlet {
    /**
     * 初始化方法，在Servlet被创建时执行，只执行一次
     *
     * @param servletConfig
     * @throws ServletException
     */
    @Override
    public void init(ServletConfig servletConfig) throws ServletException {

    }

    /**
     * 获取ServletConfig对象，ServletConfig ：Servlet的配置对象
     *
     * @return
     */
    @Override
    public ServletConfig getServletConfig() {
        return null;
    }

    /**
     * 提供服务的方法，每一次Servlet被访问时都会执行。
     *
     * @param servletRequest
     * @param servletResponse
     * @throws ServletException
     * @throws IOException
     */
    @Override
    public void service(ServletRequest servletRequest, ServletResponse servletResponse) throws ServletException, IOException {

    }

    /**
     * 获取Servlet的一些信息，例如：版本、作者等等。
     *
     * @return
     */
    @Override
    public String getServletInfo() {
        return null;
    }

    /**
     * 销毁的方法，当服务器正常关闭时执行。
     */
    @Override
    public void destroy() {

    }

}
```

#### 在web.xml中配置Servlet

```
<servlet>
    <servlet-name>ServletDemo01</servlet-name> //类名
    <servlet-class>com.stranger.servlet.ServletDemo01</servlet-class> //类路径
</servlet>

<servlet-mapping>
    <servlet-name>ServletDemo01</servlet-name>//类名
    <url-pattern>/demo01</url-pattern>//  格式 “/name”  访问名字
</servlet-mapping>
```

当服务器接受到请求之后，会解析请求的url路径，获取Servlet的资源路径，然后去查找web.xml文件，是否有对应的<url-pattern>标签体内容，然后查找类名,找到之后把字节码文件加载到内存调用方法。
![Servlet执行原理.PNG](https://img.hacpai.com/file/2019/11/Servlet执行原理-5d47a4a9.PNG)

#### 注解配置Servlet

servlet3.0及以上就可以使用直接使用注解快速配置了，不在创建web.xml

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface WebServlet {
    String name() default "";

    String[] value() default {};

    String[] urlPatterns() default {};

    int loadOnStartup() default -1;

    WebInitParam[] initParams() default {};

    boolean asyncSupported() default false;

    String smallIcon() default "";

    String largeIcon() default "";

    String description() default "";

    String displayName() default "";
}
```

直接在类上使用`@WebServlet`注解进行配置。（`@WebServlet("资源路径（访问路径）")`）

```java
@WebServlet("/demo2")
public class ServletDemo02 implements Servlet {
...
```