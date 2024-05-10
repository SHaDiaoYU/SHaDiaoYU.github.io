---
title: SpringBoot学习-配置文件
date: 2023-04-22 13:28:54
tags:
---

#### 、临时属性

#### 一、临时属性

* 指定服务运行所在的端口（格式：`--属性`）：

```xml
java -jar xxxx(项目包名).jar --server.port=8080(端口号)
```

* 如果需要设置多条指令，各个指令之间需要用空格分开，实例如下：

```xml
java -jar xxxx(项目包名).jar --server.port=8080(端口号) --spring.datasource.druid.password=123
```

**临时属性在开发环境进行测试：**

作为后端人员，需要把这些命令测通，才能交给运维人员去使用，所以该如何在开发环境下测试这些命令呢？

1. 配置运行程序，点击Edit Configurations...

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230421134224056.png" alt="image-20230421134224056" style="zoom:67%;" />

2. 在右侧菜单的 Configuration --> Environment -->Program arguments 内，配置临时属性：

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230421134604273.png" alt="image-20230421134604273" style="zoom: 50%;" />

在这里填写的命令，都会进到运行程序的 String[] args 参数中:

```java
@SpringBootApplication
public class BootApplication {

    public static void main(String[] args) {
        System.out.println(Arrays.toString(args)); // 这里会输出配置的内容如：[--server.port=8081]
        SpringApplication.run(BootApplication.class, args);// 接收args作为参数
    }

}
```

SpringApplication.run() 方法也支持不带配置参数的形式，此时无论如何进行配置，都不会生效：

```java
public static void main(String[] args) {
        System.out.println(Arrays.toString(args)); // 这里会输出配置的内容如：[--server.port=8081]
    	// 可以在启动boot程序时断开读取外部临时配置对应的入口，也就是去掉读取外部参数的形参
        SpringApplication.run(BootApplication.class);// 不带配置参数
    }
```

#### 二、配置文件等级：

根据需求的不同，可能会有不同的配置，我们可以创建不同级别的配置文件来达到灵活配置同时又避免冲突的目的。

1. 在 resources 目录（类路径）下，新创建目录 config ，在该目录下创建配置文件application.yml:

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230421142340010.png" alt="image-20230421142340010" style="zoom: 67%;" />

新的application.yml是更高级别的配置文件，新的配置文件与原来的配置文件是共同作用的，如果它们配置了相同的属性，优先级高的配置生效（高级别的覆盖低级别的配置）。

*新的问题：*有些配置，由于保密权限，我们无法得到并配置（比如数据库的权限、密码等等），并且也无法使用临时属性，这时该如何进行配置呢？

*解决：* SpringBoot给我们提供了全新的配置文件等级，来帮助我们完成不同等级的配置。

**配置文件4级分类：**

1. SpringBoot中4级配置文件

* 1级（**最高级别**）： file : config / application.yml

  说明： 在项目jar包同级目录下，有一个config目录，里面有我们的配置文件。 

* 2级：file : application.yml

  说明：在项目jar包同级目录下，有我们的配置文件。

* 3级：classPath :  config/application.yml

  说明：在项目jar包内的resources目录（类路径）下，有一个config目录，里面有我们的配置文件。

* 4级（**最低级别**）：classPath :  application.yml

  说明： 在项目jar包内的resources目录（类路径）下，有我们的配置文件。

2. 作用：

* 1级与2级留作系统打包后设置通用属性，1级常用于运维经理进行线上整体项目部署方案调控；
* 3级与4级用于系统开发阶段设置通用属性，3级常用于项目经理进行整体项目属性调控；

#### 三、自定义配置文件

配置文件一般都默认为application.yml(.properties , .yaml等等等等)，但是我们可以对配置文件名称进行自定义。

现在假设我们的配置文件有两个，一个是ebank.yml，另一个是ebank-server.yml，下面介绍如何使用我们自定义的配置文件。

* **方法1：**通过启动参数加载配置文件（无需书写配置文件扩展名）。

  格式： `--spring.config.name=配置文件名`；

  注意： yml格式与properties格式的配置文件都支持；

  <img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230422134351183.png" alt="image-20230422134351183" style="zoom:67%;" />

* **方法2：**通过启动参数加载指定文件路径下的配置文件（使用这种方式可以加载多个配置文件）

  格式：`--spring.config.location=classpath:/配置1.yml,classpath:/配置2.yml，...... `;

  注意：(a) yml格式与properties格式的配置文件都支持；

  ​		    (b) 当加载多个配置文件时，如果涉及相同的配置，则在队列中最后的配置文件中的这一项配置生效。

  <img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230422135425217.png" alt="image-20230422135425217" style="zoom:67%;" />

  如图中的两个配置文件有冲突时，后面的对应配置会生效。

**重要说明：**

* 单服务器项目：使用自定义配置文件需求较低；
* 多服务器项目：使用自定义配置文件需求较高，将所有的配置文件放在一个目录中，统一进行管理；
* 基于SpringCloud技术，所有的服务器将不再设置配置文件，而是通过配置中心进行设定，动态加载配置信息；

