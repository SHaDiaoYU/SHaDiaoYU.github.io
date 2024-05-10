---
title: SpringBoot学习-bean配置绑定
date: 2023-05-25 19:43:35
tags:
---

#### 1. 让自己定义的bean在初始化的时候自动加载配置文件中的配置：

* 配置类：

```yaml
servers:
 port: 1234
 ipAddress: 192.168.1
 timeout: -1

```

* 在我们定义的bean上使用`@ConfigurationProperties(prefix = "servers")`指定配置内容即可实现，注意设置prefix内容。

```java
@Component
@Data
@ConfigurationProperties(prefix = "servers")
public class ServerConfig {
    private String ipAdress;
    private int port;
    private long timeout;
}
```



#### 2. 如果不是自定义的bean而是给第三方bean绑定属性，又该如何实现呢？

使用druid来做示例（注意，druid只有在真正的连接数据可的时候才会进行初始化操作）：

* 在配置类中添加值，为bean注入数据：

```yaml
datasource:
  driverClassName: com.mysql.jdbc.driver123123
```

* 在配置类下添加bean，使用@ConfigurationProperties 注解并指定前缀:

```java
//定义一个第三方bean
    @Bean
    @ConfigurationProperties(prefix = "datasource")
    public DruidDataSource dataSource(){
        DruidDataSource druidDataSource = new DruidDataSource();
        return druidDataSource;
    }
```

#### 补充：`@EnableConfigurationProperties`注解

1. 可以将其理解为一个开关，其作用是标识集合中的类开启属性配置功能，可以通过配置中的属性向我们定义的类中做属性注入。

2. 所以那些在集合中的类需要使用`@ConfigurationProperties`注解关联配置文件进行配置（当然也可以不用）。

**注意：**在类上添加`@EnableConfigurationProperties(Server.class)`指定`Server.class`起作用时，springboot会自动为该类注入spring容器以进行管理。所以在使用此注解后，已经将`Server.class`类注入了，就不需要在Server类上使用@Bean或是其衍生注解将其注入容器了，否则springboot会报错。

#### 3.松散绑定

**问题引入：**

现在，我们的配置文件中的配置是这样的：

```yaml
dataSource:
  driverClassName: com.mysql.jdbc.driver123123
```

注意，我们的配置名称中使用了驼峰命名法，包含了大写字符。现在我们为已创建的bean使用此配置进行绑定，我们很容易想到，为我们的bean上添加如下注解：

```java
    @Bean
	//这里的前缀内容跟配置文件中的保持了一致，其中包含了大写字母
    @ConfigurationProperties(prefix = "dataSource") 
    public DruidDataSource dataSource(){
        ......
    }
```

在运行时却发现springboot报错了，明明我们都保持一致了，为什么还会报错呢？

![image-20230526154319821](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230526154319821.png)

观察报错内容就能够看出来为什么会报错，springboot规定prefix属性的名称必须为kebab-case(烤肉串模式)，小写的数字字母组合且必须以字母开头。

* **@ConfigurationProperties绑定属性支持属性名宽松绑定**

  在我们的配置中，springboot支持多种属性命名方式，方便地支持我们进行属性绑定。下面对ipAddress属性的配置都能够生效：

  ```yaml
  server:
   port: 1234
   #ipAddress: 192.168.1 # 驼峰命名法
   #ipaddress: 192.168.1
   ip-address: 192.168.1 # 烤肉串模式，0-0-0-0----，官方推荐使用
   #IPADDRESS: 192.168.1 
   #IP_ADDRESS: 192.168.1 # 常量命名方式
   #IP-A_D_D-RESS: 192.168.1
   
   timeout: -1
  ```

* **注意：**宽松绑定不支持注解@Value引用单个属性的方式。

#### 4. 常用计量单位使用

某些情况下，指定配置值的单位是有必要的。

* springboot支持jdk8提供的时间与空间单位：

```java
@Component
@Data
@ConfigurationProperties(prefix = "servers")
public class ServerConfig {
    private String ipAdress;
    private int port;
    private long timeout;
    // 时间单位：day
    @DurationUnit(ChronoUnit.DAYS)
    private Duration serverTimeOut;
    //空间单位： byte
    @DataSizeUnit(DataUnit.MEGABYTES)
    private DataSize dataSize;
} 
```

#### 5. bean属性校验

当我们为某些属性设置了错误的数据类型，我们希望能够获得提醒。由此引出数据校验，数据校验有助于引出系统安全性，J2EE规范中JSR303规范定义了一组有关数据校验相关的API。

**使用：**

（1）第一步，在pom文件中引入依赖，导入JSR303规范坐标（这是接口）与Hibernate校验框架对应坐标（这是实现）：

```xml
<dependency>
	<groupId>javax.validation</groupId>
	<artifactId>validation-api</artifactId>
</dependency>
<dependency>
	<groupId>org.hibernate.validator</groupId>
	<artifactId>hibernate-validator</artifactId>
</dependency>
```

（2）对bean开启校验功能：

```java
@Component
@Data
@ConfigurationProperties(prefix = "servers")
@Validated
public class ServerConfig {
   ......
} 
```

（3）设置校验规则：

```java
@Component
@Data
@ConfigurationProperties(prefix = "servers")
@Validated
public class ServerConfig {
   ......
   @Max(value= 8888,message="最大值不能超过8888")
   private int port;
} 
```

#### 6. 进制数据转换规则

由问题引出：现在有这样一条配置：

```yaml
config:
 password: 0127
```

ok，现在我们获取它：

```java
@Value("${config.password}")
private String password;
```

将其输出，得到的值却是**87** ，将0127用双引号包起来，输出会正常，为什么会发生这样的事情呢？

* 原因在于yaml支持八进制值，而八进制值的书写要求是**以0开头，最大数字不超过7** 。而我们的设置内容刚好满足这条规则，同时，yaml支持字符串直接书写，可以不用双引号包裹，于是0127被认为是八进制数值进行解析，其输出为十进制的87。

这里我们不得不重新回顾一下yaml语法规则中的字面值表达方式了：

![image-20230529140807338](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230529140807338.png)

```yaml
二进制: 0b1001_1010
八进制: 07237_1224
十六进制: 0x123A_DCFE
```





