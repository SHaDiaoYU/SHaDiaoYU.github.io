---
title: SpringBoot学习-SQL相关内容
date: 2023-05-30 16:44:35
tags:
---

#### 1. 数据源配置

SpringBoot提供了3种内嵌的数据源对象供开发者选择

* HikariCP：默认的内置数据源对象；
* Tomcat提供DataSource：HikariCP不可用的情况下，且在web环境中，将使用的tomcat服务器配置的数据源对象
* Commons DBCP：Hikari不可用，tomcat数据源也不可用，将使用dbcp数据源；

通用配置无法设置具体的数据源配置信息，仅提供基本的连接相关配置，如需配置，在下一级配置中设置具体设定：

```yaml
spring:
  datasource:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/test_mybatis
      username: root
      password: root
      hikari:
       maximum-pool-size: 50
```

