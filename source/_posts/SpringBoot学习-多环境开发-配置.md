---
title: SpringBoot学习-多环境开发
date: 2023-04-22 14:06:25
tags:
---

#### 一、引入

一个项目在开发过程中会经历多个环境，包括开发环境、测试环境、生产环境等等，而在这不同的环境中，对于某些数据的配置又会有所区别。所以需要保证我们在开发的时候是一个环境，部署到线上时又能切换到另外的环境了。

#### 二、具体操作（yaml版本）

第一步，设置环境，不同的环境之间用`---`作分隔：

```yaml
#应用环境
spring:
  profiles:
    active: pro
    
#公共配置

......

---
#设置环境
#生产环境
spring:
  profiles: pro
server:
  port: 8081
---
#开发环境
spring:
  profiles: dev
server:
  port: 8082
---
#测试环境(标准格式)
spring:
  config: 
   activate:
  	on-profile: test
server:
  port: 8083
```

注意：

* 在设置环境部分，我们通过为每一个环境指定spring.profiles属性的值来为我们的环境配置命名；
* 在应用环境部分，我们通过设置spring.profiles.active属性的值来指定我们使用哪一个环境的配置；

**这种操作的缺陷：**

​	各种环境的配置都放在了一个配置文件内，有暴露配置的风险（比如敏感的数据库密码等等），因此需要一种新的解决方式。

**多文件配置：**

使用多文件配置，将不同环境的配置以及公共配置独立成为不同的文件，并分别进行配置：

1. 主启动配置文件 application.yml

```yaml
spring:
 profiles:
  active: dev
```

2. 环境分类配置文件 application-pro.yml （注意格式：application**-类型**.yml）

```yaml
server:
 port: 8080
......
```

3. 环境分类配置文件 application-dev.yml

```yaml
server:
 port: 8081
......
```

4. 环境分类配置文件 application-test.yml

```yaml
server:
 port: 8082
......
```

注意：

* 主配置文件中设置公共配置（全局）；
* 环境分类配置文件中常用于设置冲突属性（局部）；

#### 三、具体操作（properties版本）

使用多文件配置，将不同环境的配置以及公共配置独立成为不同的文件，并分别进行配置：

1. 主启动配置文件 application.properties

```properties
spring.profiles.active=dev
```

2. 环境分类配置文件 application-pro.properties（注意格式：application**-类型**.yml）

```properties
server.port=8080
......
```

3. 环境分类配置文件 application-dev.properties

```properties
server.port=8081
......
```

4. 环境分类配置文件 application-test.properties

```properties
server.port=8082
......
```

#### 四、信息拆分&独立

* 根据功能对配置文件中的信息进行拆分，并制作成独立的配置文件，命名规则如下：

  * application-devDB.yml

  * application-devRedis.yml

  * application-devMVC.yml

    .....

**方法一：**

使用include属性在激活指定环境的情况下，同时对多个环境进行加载使其生效，多个环境之间用逗号分隔。

```yaml
spring:
 profiles:
  active: dev
  include: devDB,devRedis,devMVC
```

注意：include之中的**顺序**问题，如果涉及相同的配置，则顺序靠后的配置生效。

缺点：如果需要切换环境，则需要重新写一遍include中的内容，很繁琐。

**方法二：**

使用group属性定义多种主环境与子环境的包含关系，格式如下：

```yaml
spring:
 profiles:
  active: dev
  group:
   "dev": devDB,devRedis
   "pro": proDB,proMVC
```

**注意：** 

* 当主环境dev与其他环境有相同属性时，主环境生效；
* 其他环境中有相同属性时，最后加载的环境属性生效；



