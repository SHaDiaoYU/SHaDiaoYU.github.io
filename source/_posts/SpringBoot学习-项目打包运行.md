---
title: SpringBoot学习-项目打包运行
date: 2023-04-15 14:36:53
tags:
---

#### Windows系统下进行项目打包运行：

#### 一、Windows系统下进行项目打包运行：

**常规方式：**

1. 在 maven --> Lifecycle-->package下，双击package进行打包：

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/meaven.png" style="zoom:67%;" />

​	打包完毕后，在项目左侧target目录下可观察到打包完成的jar包：

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/jar.png" style="zoom:67%;" />

2. 点击右键菜单-->show in explorer，进入文件目录，在该目录下打开cmd，输入指令：`java -jar 包名.jar`即可运行。

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/cmd.png" style="zoom:67%;" />

​	运行后可观察到启动过程：

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/run.png" style="zoom:50%;" />

**需要注意的点：**如果使用默认的打包运行操作，会运行test下的所有测试用例（在实际运行过程中并不需要），因此需要跳过测试：

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/test.png" style="zoom: 67%;" />

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230415145838195.png" alt="image-20230415145838195" style="zoom: 67%;" />

同时，需要注意，打包需要引入spring的maven插件，否则无法进行打包：

```xml
<plugin>
     <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-maven-plugin</artifactId>
             <config  uration>
                 <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
```

在打包过程中，我们也可能会遇到端口被占用的问题，这个时候就需要我们手动结束占用该端口的进程,具体的操作指令如下：

<img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230415151857372.png" alt="image-20230415151857372" style="zoom: 33%;" />
