---
title: SpringMVC拦截器
date: 2023-03-06 16:46:29
tags:
---

#### SpringMVC拦截器：

SpringMVC中的拦截器用于拦截控制器方法的执行

SpringMVC中的拦截器需要实现HandlerInterceptor或者继承HandlerInteceptorAdapter类

HandleInterceptor类下有三个方法：

* preHandle:在请求控制器前执行；
* postHandle:在控制器执行之后，页面渲染前执行；
* afterHandle：在页面渲染之后执行；

#### SpringMVC拦截器配置：

SpringMVC的拦截器必须在SpringMVC中的配置文件进行设置（3种方式）：

```xml
<!--配置拦截器-->
    <mvc:interceptors>
        <!--1. 这样会拦截所有请求-->
        <!--<bean class="com.jiawei.mvc.interceptors.FirstInterceptor"></bean>-->
        <!--2. 这种方式同样会拦截所有请求，需要注册拦截器为组件-->
        <!--<ref bean="firstInterceptor"></ref>-->
        <!--3. 使用这种方式可以配置拦截器拦截规则-->
        <mvc:interceptor>
            <!--配置拦截路径：拦截一层请求就是/*，拦截所有请求就是/**-->
            <mvc:mapping path="/**"/>
           <!-- 配置放行路径：-->
            <mvc:exclude-mapping path="/"/>
           <!-- 指定拦截器，可以使用<bean>标签，也可以使用<ref>标签-->
            <ref bean="firstInterceptor"></ref>
        </mvc:interceptor>
    </mvc:interceptors>
```

#### 多个拦截器的执行顺序：

以在SpringMVC.xml中配置的拦截器顺序为标准顺序:

* 若每个拦截器的preHandle方法的返回值都为true，此时多个拦截器的执行顺序与标准顺序有关：
  * preHandle执行的顺序为**标准顺序**，postHandle和afterComplation方法**反序**执行。
* 若某个拦截器的preHandle方法的返回值为false：
  * 返回false的拦截器与它之前的拦截器的preHandle方法都会执行，所有的postHandle方法都不执行，返回false的拦截器之前的拦截器的afterComplation方法会执行。



