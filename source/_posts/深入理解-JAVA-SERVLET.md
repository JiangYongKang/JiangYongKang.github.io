---
title: 深入理解 Java Servlet
date: 2020-11-10 10:24:42
tags: Servlet
categories: Java
---

### Servlet 简介

`Servlet（Server Applet）`，全称 `Java Servlet`。是用 Java 编写的服务器端程序。其主要功能在于交互式地浏览和修改数据，生成动态 Web 内容。狭义的 Servlet 是指 Java 语言实现的一个接口，广义的 Servlet 是指任何实现了这个 Servlet 接口的类，一般情况下，人们将 Servlet 理解为后者。Servlet 运行于支持 Java 的应用服务器中。从实现上讲，Servlet 可以响应任何类型的请求，但绝大多数情况下 Servlet 只用来扩展基于 HTTP 协议的 Web 服务器。最早支持 Servlet 标准的是 JavaSoft 的 Java Web Server 。此后，一些其它的基于 Java 的 Web 服务器开始支持标准的 Servlet。

<!-- more -->

### 如何实现 Servlet

首先 Servlet 的接口并不在 JDK 里面，我们需要引入 `servlet-api` 这个第三方库才可以找到对应的接口。下面分别是 Servlet2.x Servlet3.x Servlet4.x 最新稳定版本的依赖，这三个大版本之间有什么特性更新会在最后的地方说明。

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>servlet-api</artifactId>
    <version>2.5</version>
    <scope>provided</scope>
</dependency>
```

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>3.1.0</version>
    <scope>provided</scope>
</dependency>
```

```xml
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>javax.servlet-api</artifactId>
    <version>4.0.1</version>
    <scope>provided</scope>
</dependency>
```

`Servlet` 需要继承 `HttpServlet`，然后实现父类用于处理请求的方法，如：`doGet()` `doPost()` `service()` 方法等，这里我们创建一个 `HelloServlet`。

```java
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.setContentType("text/html");
        PrintWriter writer = resp.getWriter();
        writer.write("<h1>Hello " + req.getParameter("name") + "</h1>");
        writer.flush();
    }
}
```

Servlet 容器会在启动的时候读取 `web.xml` 文件，找到文件中配置的 Servlet 信息，进行反射和实例化。所以还需要在 web.xml 中进行配置。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE web-app
        PUBLIC "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
        "http://java.sun.com/dtd/web-app_2_3.dtd">
<web-app>
    <servlet>
        <servlet-name>Controller</servlet-name>
        <servlet-class>com.servlet.learning.HelloServlet</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>Controller</servlet-name>
        <url-pattern>*.do</url-pattern>
    </servlet-mapping>
</web-app>
```

将上面的代码打包为 war 格式，部署至 tomcat，访问 [http://localhost:8080/JavaServletLearning/HelloServlet.do?name=nickname](http://localhost:8080/JavaServletLearning/HelloServlet.do?name=nickname)

### 如何理解 Servlet

狭义的理解 Servlet 是 `javax.servlet-api` 包里面的一个接口 `javax.servlet.Servlet`，这个接口规范了应用在处理网络请求时必须做的一些事情，如初始化，处理业务逻辑，销毁。下面是 `Servlet` 的源码注释

```java
package javax.servlet;
import java.io.IOException;

public interface Servlet {

    // 处理业务逻辑之前的初始化动作
    public void init(ServletConfig config) throws ServletException;

    // 获取 Servlet 的配置信息
    public ServletConfig getServletConfig();

    // 执行业务逻辑，Servlet 容器会将网络请求解析掉，封装成 ServletRequest 对象传递进来
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException;

    public String getServletInfo();

    // 处理业务逻辑之后的销毁动作
    public void destroy();
}
```

为什么要定义这样的一个接口？Servlet 接口定义的是一套处理网络请求的规范，所有 Servlet 的实现类都必须实现这几个方法，而所有支持 Servlet 的 Web 服务器都会调用这五个方法。

以 Tomcat 为例，Tomcat 是一个 Servlet 容器，当浏览器发出一个请求的时候，这个请求首先会被 Tomcat 接收，根据请求的 URL 来决定由哪个 Servlet 实现类来处理，然后依次调用 Servlet 实现类的 init() service() destroy() 方法，最后将响应结果解析并返回给浏览器。

其它的 Servlet 容器如 Jetty Jboss 也都是这样进行处理的，这也是为什么同样的一套代码，可以随意部署在不同的支持了 Servlet 协议的 Web 服务器中。

在 Java Servlet API 中已经提供了两个抽象类方便开发者去实现自己的 Servlet，分别是 `javax.servlet.GenericServlet` 和 `javax.servlet.http.HttpServlet`。`GenericServlet` 定义的是一个通用的和具体网络协议无关的 Servlet，而 `HttpServlet` 则定义了 Http 的 Servlet，我们在开发 Web 应用时通常继承 `HttpServlet` 来定义自己的业务逻辑，SpringMvc 中的 `DispatcherServlet` 也是继承于 `HttpServlet` 并进行了二次封装。

```java
package javax.servlet;

import java.io.IOException;
import java.util.Enumeration;
import java.util.ResourceBundle;

public abstract class GenericServlet implements Servlet, ServletConfig, java.io.Serializable {

    private transient ServletConfig config;

    public GenericServlet() { }

    public void init() throws ServletException {
    }

    // 初始化的时候设置 ServletConfig 对象
    public void init(ServletConfig config) throws ServletException {
        this.config = config;
        this.init();
    }

    // GenericServlet 并没有实现 service 方法，把实现留个了下面的子类去做
    public abstract void service(ServletRequest req, ServletResponse res) throws ServletException, IOException;

    // 这个方法并不是销毁 Servlet 对象，而是在正在销毁 Servlet 对象前必然调用的一个方法，我们可以重写这个方法做一些回收的动作
    public void destroy() {
    }
}
```

```java
package javax.servlet.http;

import java.io.IOException;
import java.io.PrintWriter;
import java.io.OutputStreamWriter;
import java.io.UnsupportedEncodingException;
import java.lang.reflect.Method;
import java.text.MessageFormat;
import java.util.Enumeration;
import java.util.Locale;
import java.util.ResourceBundle;

import javax.servlet.GenericServlet;
import javax.servlet.ServletException;
import javax.servlet.ServletOutputStream;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;

public abstract class HttpServlet extends GenericServlet implements java.io.Serializable {

    // 我们在实现自己的业务逻辑的时候，一般不建议重写这个方法，因为它已经做了很多处理
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
    	HttpServletRequest	request;
    	HttpServletResponse	response;
    	try {
    	    request = (HttpServletRequest) req;
    	    response = (HttpServletResponse) res;
    	} catch (ClassCastException e) {
    	    throw new ServletException("non-HTTP request or response");
    	}
    	service(request, response);
    }

    // 真正的处理逻辑再这里
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String method = req.getMethod(); // 获取http请求类型

        if (method.equals(METHOD_GET)) {
            long lastModified = getLastModified(req);
            if (lastModified == -1) {
                doGet(req, resp);
            } else {
                long ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
                if (ifModifiedSince < (lastModified / 1000 * 1000)) { // 判断资源是否被修改
                    maybeSetLastModified(resp, lastModified);
                    doGet(req, resp);
                } else {
                    resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED); // 如果没被修改，直接返回 304
                }
            }

        // 下面就是根据不同的请求类型调用不同的方法
        } else if (method.equals(METHOD_HEAD)) {
            long lastModified = getLastModified(req);
            maybeSetLastModified(resp, lastModified);
            doHead(req, resp);
        } else if (method.equals(METHOD_POST)) {
            doPost(req, resp);
        } else if (method.equals(METHOD_PUT)) {
            doPut(req, resp);
        } else if (method.equals(METHOD_DELETE)) {
            doDelete(req, resp);
        } else if (method.equals(METHOD_OPTIONS)) {
            doOptions(req,resp);
        } else if (method.equals(METHOD_TRACE)) {
            doTrace(req,resp);
        } else {
            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[1];
            errArgs[0] = method;
            errMsg = MessageFormat.format(errMsg, errArgs);
            resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg); // 匹配不到请求类型，返回 504
        }
    }
}
```

### 在 Servlet 中使用过滤器

Servlet 过滤器可以动态地拦截请求和响应，以变换或使用包含在请求或响应中的信息。可以将一个或多个 Servlet 过滤器附加到一个 Servlet 或一组 Servlet。Servlet 过滤器也可以附加到 JavaServer Pages (JSP) 文件和 HTML 页面。调用 Servlet 前调用所有附加的 Servlet 过滤器。Servlet 过滤器是可用于 Servlet 编程的 Java 类，可以实现以下目的：

-   在客户端的请求访问后端资源之前，拦截这些请求。
-   在服务器的响应发送回客户端之前，处理这些响应。
    首先需要实现 `Filter` 接口，并实现它的三个方法：

```java
public class BeforeFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("BeforeFilter 初始化...");
    }
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("BeforeFilter 执行逻辑...");
        chain.doFilter(request, response);
    }
    @Override
    public void destroy() {
        System.out.println("BeforeFilter 被销毁...");
    }
}
```

与 Servlet 不同的是，过滤器会在容器启动的时候就被初始化，而不是接收到请求了才初始化。现在，我们把过滤器配置到 web.xml 中：

```xml
<web-app>
    <!-- 过滤器的执行顺序与配置顺序一致，一般把过滤器配置在所有的 Servlet 之前 -->
    <filter>
        <filter-name>BeforeFilter</filter-name>
        <filter-class>com.servlet.learning.BeforeFilter</filter-class>
    </filter>
    <!-- 过滤器的路径拦截规则 -->
    <filter-mapping>
        <filter-name>BeforeFilter</filter-name>
        <url-pattern>*.do</url-pattern>
    </filter-mapping>
</web-app>
```

容器启动时会读取 web.xml 文件，根据类的完全限定名去初始化过滤器。需要注意的是，在执行完 doFilter() 方法中的逻辑后，一定要调用 chain.doFilter(request, response) 方法，将请求传回过滤器链。

### Servlet 的生命周期

Servlet 的生命周期主要包括加载实例化、初始化、处理客户端请求、销毁。加载和实例化主要由 Web 容器完成，后面的三个步骤则分别由对象的 init() service() destroy() 方法提供。

初始化：init() 这个方法只会在创建 Servlet 对象时被调用一次，主要用来处理一些初始化的事情。Servlet 对象会在什么时候初始化呢？1. 容器启动时会自动初始化一些在 web.xml 中标记了 `<loadstartup>1</loadstartup>` 的 Servlet。2. Servlet 启动后，首次接收到请求的时候。3. Servlet 类文件被更新后。

处理客户端请求：Servlet 容器调用 service() 方法来处理来自客户端（浏览器）的请求，并把相应结果返回给客户端。每次 Servlet 容器接收到一个 Http 请求，Servlet 容器会产生一个新的线程并调用 Servlet 实例的 service 方法，service 方法会检查 HTTP 请求类型（GET、POST、PUT、DELETE 等），并在适当的时候调用 doGet、doPost、doPut、doDelete 方法。所以，在编码请求处理逻辑的时候，我们只需要关注 doGet()、或 doPost() 的具体实现即可。

销毁：destroy() 方法只会被调用一次，在 Servlet 生命周期结束时被调用。destroy() 方法可以让您的 Servlet 关闭数据库连接、停止后台线程、把 Cookie 列表或点击计数器写入到磁盘，并执行其他类似的清理活动。

### 什么是 Servlet 容器？

Servlet 容器指的是支持 Servlet 协议的 Web 服务器，容器会监听端口，接收请求。将接收到的报文转换为 ServletRequest 对象传递给 Servlet 对象，等待 Servlet 对象处理完逻辑后，把返回的 ServletResponse 对象 组装成响应报文返回给客户端。

### Servlet 是线程安全的吗？

Servlet 体系结构是建立在 Java 多线程机制之上的，它的生命周期是由 Web 容器负责的。当客户端第一次请求时，容器会根据 web.xml 去实例化对应的 Servlet，后续的请求线程进来后不会再实例化新的 Servlet 对象，也就是会有多个线程在使用 Servlet 对象。这样，当两个或多个线程同时访问同一个 Servlet 时，可能会发生多个线程同时访问同一资源的情况，数据可能会变得不一致。所以在用 Servlet 构建的 Web 应用时如果不注意线程安全的问题，会使所写的 Servlet 程序有难以发现的错误。那么如何编写线程安全的 Servlet 呢？主要由以下三种方式：

1. 实现 SingleThreadModel 接口。该接口指定了系统如何处理对同一个 Servlet 的调用。如果一个 Servlet 被这个接口指定,那么在这个 Servlet 中的 service 方法将不会有两个线程被同时执行，当然也就不存在线程安全的问题。这种方法只要将前面的 HelloServlet 类的定义修改为：`public class HelloServlet extends HttpServlet implements SingleThreadModel`。这种方法在 Servlet 2.4 以后就不推荐使用了，因为这样做会使每一个请求都去创建 Servlet 对象，牺牲空间去换安全的做法，会带来很大的系统开销。
2. 使用 `synchronized` 关键字手动同步
3. 避免使用实例变量，线程安全的问题是多个线程同时读写共享数据造成的。所以只要不在 service() 方法中使用实例变量，改为使用局部变量，这种方法是最方便而且开销最小的。

### 参考文章

-   [https://www.zhihu.com/question/21416727](https://www.zhihu.com/question/21416727)
-   [https://www.runoob.com/servlet/servlet-tutorial.html](https://www.runoob.com/servlet/servlet-tutorial.html)
-   [https://www.liaoxuefeng.com/wiki/1252599548343744/1255945497738400](https://www.liaoxuefeng.com/wiki/1252599548343744/1255945497738400)

本文代码仓库：[https://github.com/JiangYongKang/JavaServletLearning](https://github.com/JiangYongKang/JavaServletLearning)
