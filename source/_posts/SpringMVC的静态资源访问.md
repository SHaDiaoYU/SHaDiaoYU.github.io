---
title: SpringMVC的静态资源访问
date: 2023-03-02 15:50:02
tags:
---

在web阶段的时候，静态资源是交给默认的servlet来处理的。

具体位置可在tomcat中的web.xml中发现。

#### tomcat中的web.xml与我们工程中的web.xml的关系：

二者相当于是一种继承关系，tomcat下的web.xml作用于部署在tomcat下的所有工程上，而我们工程中的web.xml只作用于当前工程。

如果当前工程中的配置与tomcat的配置不同，以当前工程的配置为准。

dispatcherServlet处理请求方式：每一次都把请求地址去控制器中寻找相对应的请求映射，但是控制器中没有写访问静态资源的请求映射时，就会找不到静态资源然后报404错误。

所以在遇到静态资源404的情况时，需要我们在当前项目的springMVC.xml中进行如下配置：

```java
<!--开放对静态资源的访问-->
<mvc:default-servlet-handler></mvc:default-servlet-handler>
```

如果浏览器的请求dispatcherServlet访问不了，在进行此配置之后，就会交给tomcat的defaultServlet来处理。

使用该配置时同样需要配置MVC注解驱动:

```java
<!--开启注解驱动，之后很多场景，都会用到-->
<mvc:annotation-driven></mvc:annotation-driven>
```

如果不进行这项配置，我们所有的请求都会被dispatcherServlet所处理

#### 总结一下，所有的请求先由diapatcherServlet处理，如果处理不了就交给defaultServlet来处理，如果仍处理不了就报404错误了。