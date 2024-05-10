---
title: SpringBoot学习--配置绑定
date: 2023-04-03 11:16:46
tags:
---

#### 需求场景：

有这样的一个*javaBean* ，其内容如下：

```java
public class Car {

    private String brand;
    private Integer price;

    //getter、setter、toString()等方法已省略
}
```

在配置文件application.properties内，有这样的两条配置：

```properties
myCar.brand="benz"
myCar.price=1000
```

现在需要读取properties文件的内容，并且把它封装到*javaBean*内，如果使用普通的方式实现，需要用IO流来读取跟数据有关的配置（配置文件中的配置信息可能有很多，因此读取起来可能会很耗时间）再完成封装。这个过程会很麻烦。SpringBoot给我们提供了实现配置绑定的几种方式，以下作简单介绍。

####  SpringBoot实现配置绑定的几种方式：

1. 使用**@ConfigurationProperties**注解

   * 位置：在javaBean--Car内，使用注解**@ConfigurationProperties** ，并设置**prefix**属性为 “myCar”,该属性会将组件中的属性与配置文件中该前缀的属性配置绑定;

   * 使用**@Component**注解，标识当前javaBean为组件，将当前的javaBean（组件）添加至容器内，因为只有在容器中的组件才能使用SpringBoot的强大功能；

   示例如下：

   ```java
   @Component
   @ConfigurationProperties(prefix = "mycar")
   public class Car {
   
       private String brand;
       private Integer price;
   	......
   ```

   查看是否完成了配置绑定：

   ```java
   @RestController
   public class HelloController {
       // 利用spring提供的自动注入，将容器中的组件car拿过来就行
       @Autowired
       Car car;
       
       @RequestMapping("/car")
       public Car car(){
           return car;
       }
   }
   ```

   2. 使用**@EnableConfigurationProperties**注解

      * 位置：被**@Configuration**标识过的配置类中，使用**@EnableConfigurationProperties**注解，并设置参数为**(javaBean)xxx.class**
      * 即可为javaBean开启属性配置绑定功能，同时为javaBean自动注册到容器中（这样的话，就不需要再为javaBean添加@Component注解了）。
      * 适用的场景：在使用第三方包的时候，不便于在类上设置@Component注解时，便可使用此方法在配置类上进行操作。

      示例：

      ```java
      @EnableConfigurationProperties(Car.class)
      @Configuration(proxyBeanMethods = true)
      public class Myconfig {
      	......
      }
      ```

      

