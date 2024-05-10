---
title: SpringBoot学习-多环境开发控制
date: 2023-04-27 15:39:44
tags:
---

#### 一、引言

现在maven和spring都有多环境的开发控制（profile），假设现在是这样的情景：maven中设置的是生产环境，springboot中设置的是开发环境。究竟哪一个有效呢？或者说如果发生冲突了，谁生效？

#### 二、多环境开发控制

springboot是依赖maven坐标的配置进行工作的，没有maven，spring就无法工作了，所以maven一定是先于spring在运行。如果现在双方都有配置，一定是以maven配置为主，springboot的配置为辅。下面说明如何通过maven的配置来进行多环境开发控制。

**操作：**

* 在pom.xml 中设置多环境，其中通过设置`<activeByDefault>`是否为真，来决定默认启用哪个环境。

```xml
<!--    设置多环境-->
    <profiles>
        <profile>
            <id>env_dev</id>
            <properties>
                <profile.active>dev</profile.active>
            </properties>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
        <profile>
            <id>env_pro</id>
            <properties>
                <profile.active>pro</profile.active>
            </properties>
        </profile>
    </profiles>
```

* 同时springboot中关于多环境的配置也需要修改，使用@符引用pom.xml中的配置profile.active

```yaml
spring:
 profiles:
  active: @profile.active@
  group:
   "dev": devDB,devRedis
   "pro": proDB,proMVC
```

**注意：**如果在切换生产环境中发现即使设置了pom.xml`<activeByDefault>`为真后，环境依然没有切换，此时需要点击maven下的compile进行手动编译方可生效。这是idea的bug。

