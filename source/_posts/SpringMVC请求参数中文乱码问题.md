---
title: SpringMVC请求参数中文乱码问题
date: 2022-11-19 12:57:26
tags:
---

# SpringMVC请求中文参数乱码的问题

## 讲解：尚硅谷SpringMVC P32

* get请求不会发生乱码，get请求的乱码是由tomcat造成的，解决需要修改tomcat/conf/server.xml，可以一次性解决。
* 使用post方式时如果出现乱码，在控制器具体方法内使用characterEndocing设置请求编码无法解决问题，因为获取请求参数先发生，在请求方法里设置编码是无效的。

### 解决办法：

* 在servlet执行之前完成参数的编码。（）

* 联想服务器中三大组件：监听器、过滤器、servlet。其中servletContextListener加载时间最早，其次是过滤器、最后是servlet。

* 对于监听器servletContextListener，由于其在整个生命周期内只执行一次，所以用来解决多个请求的编码是不合适的。

* 对于过滤器，只要满足请求路径，就会对请求进行过滤，因此考虑使用过滤器解决编码问题。

* 修改web.xml，添加过滤器：

  ```java
  <filter>
      <filter-name>CharacterEncodingFilter</filter-name>
      <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>
      <init-param>
        <param-name>encoding</param-name>
        <param-value>utf-8</param-value>
      </init-param>
    </filter>
    <filter-mapping>
      <filter-name>CharacterEncodingFilter</filter-name>
      <url-pattern>/*</url-pattern>
    </filter-mapping>
  ```

  注：SpringMVC中处理编码的过滤器一定要配置到其他过滤器之前，否则无效。