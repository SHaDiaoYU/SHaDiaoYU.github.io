---
title: SpringBoot学习-日志
date: 2023-04-27 19:25:33
tags:
---

#### 一、日志的作用

* 编程期调试代码
* 运营期记录信息
  * 记录日常运营重要信息（峰值流量、平均响应时长。。。）
  * 记录应用报错信息（错误堆栈）
  * 记录运维过程数据（扩容、宕机、报警。。。）

#### 二、使用日志

日志的使用分为两部分：(1) 创建记录日志的对象、(2) 手工的记录日志

* 创建记录日志的对象：

```java
// 1、创建记录日志的对象
    private static final Logger log = LoggerFactory.getLogger(UserController.class);
```

* 手工记录日志

```java
@GetMapping("/users")
    public List<User> getAllUser() {
        log.info("the information");
        log.error("the error");
        log.debug("debug...");
        log.warn("the warning");
        return service.list();
    }
```

**日志级别： error > warning > info >debug**

**开启debug模式**

项目启动后，默认显示的日志内容为info及以上级别的，如果需要显示debug日志，则需要手动开启debug模式

方式1：在program arguments 内部添加 `--debug`;

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230427194247750.png" alt="image-20230427194247750" style="zoom:67%;" />

方式2： 在配置文件application.yml中添加属性`debug: true`（以上两种方式不推荐使用）

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230427194523699.png" alt="image-20230427194523699" style="zoom:67%;" />

方式3： 在配置文件application.yml中，进行配置，这种方式的好处就是可以定义显示的日志级别：

```yaml
logging:
  level:
    root: debug
```

#### 三、设置日志级别

```yaml
logging:
  # 设置分组
  group:
   ebank: com.jiawei.boot.service,com.jiawei.boot.controller
   iservice: com.alibaba
  level:
    root: info
    # 1. 设置某个包的日志级别(不推荐，很低效)
    com.jiawei.boot.service: debug
    # 2. 设置分组，对某个分组设置日志级别
    ebank: warn
```

#### 四、快速创建日志对象

在每个类上创建记录日志的对象，很冗余。下面介绍快速创建日志对象的方式：

方法1：使用继承类，创建一个Basement.java，在使用的类上继承Basement即可。

```java
public class Basement {
    private Class clazz;
    private static Logger log;
    
    public Basement() {
        clazz = this.getClass();
        log = LoggerFactory.getLogger(clazz);
    }
}
```

方法2：使用`@Slf4j`注解，注释在类上，使用`log.info(xxx)`使用；

#### 五、日志输出格式控制

**日志输出格式**

![image-20230427202254834](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230427202254834.png)

* PID： 进程ID，用于表明当前操作所属的进程，当多服务同时记录日志时，该值课用于协助程序员调试程序。
* 所属类/接口名： 当前显示信息为SpringBoot重写后的信息，名称过长时，简化包名书写为首字母，甚至直接删除。

**日志输出格式控制**

在配置文件中设置日志输出格式：

```yaml
logging:
 group:
 pattern:
  # 自定义的日志输出格式 %d:日期，%p:级别，%m:消息，%n:换行
  console: "%d %5p - %m %n"
  # 自定义颜色，以及输出展位控制.5p意味着该信息占据5个空格,%t代表线程名,%c代表类名,-40.40c意味着左对齐40个字符的位置
  console: "%d %clr(%5p) --- [%16t] %clr(%-40.40c){color} : %m %n"
```

#### 六、文件记录日志

* 使用文件记录日志信息，在配置文件中进行配置即可:

```yaml
logging:
 file:
  name: server.log
```

日志文件server.log位于工程目录下。

* 设置日志文件记录日志大小，超出这个大小就会创建新的日志文件。

```yaml
logging:
 logback:
  rollingpolicy:
   # 设置日志文件大小限制
   max-file-size: 4KB
   #设置日志文件名称格式( %d即date,%i即index )：下面格式的示例为：server.2020-01-01.0.log 
   file-name-pattern: server.%d{yyyy-MM-dd}.%i.log
```





