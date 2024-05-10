---
title: SpringCloud学习笔记
date: 2023-10-16 15:27:59
tags:
---

### 1. Feign与Gateway中的负载均衡：

* Feign中的负载均衡是指微服务的实例和实例之间互相调用时进行负载均衡（内部）；
* Gateway中的负载均衡是指对用户请求进行负载均衡（外部）；

### 2. Docker命令

####  （1）Docker镜像命令：

* 查看镜像：

  ```
  docker images
  ```

* 拉取镜像

  ```
  docker pull
  ```

* 输出镜像

  ```
  docker save -o [outputFile] [imageName]
  ```

* 删除镜像

  ```
  docker rmi [imageName]
  ```

* 从文件加载镜像

  ```
  docker load -i [xxx.tar(镜像文件)]
  ```

#### （2）Docker容器命令：

* 运行容器

  ```
  docker run --name [containerName] -p [80]:[80] -d [imageName]
  ```

  命令解读：

  * docker run: 创建并运行一个容器；
  * --name: 给容器起一个名字；
  * -p: 将宿主机端口与容器端口映射，冒号左侧是宿主机端口，右侧是容器端口；
  * -d: 后台运行容器；
  * 【imageName】: 镜像名称；

* 暂停容器（运行态到暂停状态）

  ```
  docker pause
  ```

* 停止暂停（暂停状态到运行状态）

  ```
  docker unpause
  ```

* 终止容器（运行态到终止态）

  ```
  docker stop
  ```

* 启动容器（终止态到运行态）

  ```
  docker start
  ```

* 进入容器执行命令

  ```
  docker exec -it [containerName] bash
  ```

  命令解读：

  * docker exec：进入容器内部，执行一个命令；
  * -it：给当前进入的容器创建一个标准输入、输出终端，允许我们与容器进行交互；
  * mn：要进入的容器的名称；
  * bash：进入 容器后执行的命令，bash是一个linux终端的交互命令；

* 查看容器运行日志

  ```
  docker logs
  ```

* 查看所有运行的容器及状态

  ```
  docker ps
  ```

* 删除指定容器

  ```
  docker rm
  ```


#### （3）Docker数据卷命令

* 数据卷操作的基本语法：

  ```java
  docker volume [COMMAND]
  ```

  docker volume命令是数据卷操作，根据命令后跟随的command来确定下一步的操作：

  * create：创建一个volume;
  * inspect：显示一个或多个volume的信息；
  * ls：列出所有volume;
  * prune：删除未使用的volume;
  * rm：删除一个或多个指定的volume;
  
* 例子：创建一个nginx容器，并挂载数据卷到容器内的html目录：

  ```
  docker run --name myNginx -v html:/usr/share/nginx/html -p 80:80 -d nginx
  ```

### 3. Docker自定义镜像

#### 镜像结构：

* 镜像是将应用程序及其所需要的系统函数库、环境、配置、依赖打包而成。

#### DockerFile:

DockerFile就是一个文本文件，其中包含 一个个的**指令（Instruction）**，用指令说明要执行什么操作来构建镜像。每一个指令都会形成一层layer。

![image-20231021202201010](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20231021202201010.png)

根据dockerFile创建镜像

```
docker build -t [repository:tag] [dockFilePath]
```

例子：

```
docker build -t javaweb:1.0 ../dockerfile/
```

* DockerFile的本质是一个文件，通过指令描述镜像的构建过程。
* DockerFile的第一行必须是FROM，从一个基础镜像来构建
* 基础镜像可以是基本操作系统，如Ubuntu。也可以是其他人制作好的镜像，例如java8-alpine。

# 微服务架构特征

* 单一职责：微服务拆分粒度更小，每个业务都对应唯一的业务能力，做到单一职责。
* 自治：团队独立，技术独立，数据独立，独立部署和交付。
* 面向服务：服务提供统一标准的接口，与语言和技术无关。
* 隔离性强：服务调用做好隔离，容错，降级、避免出现级联问题。

## 服务拆分原则

* 不同微服务，不要重复开发相同的业务
* 微服务数据独立，不要访问其它微服务的数据库
* 微服务可以将自己的业务暴露为接口，供其他微服务调用

## 1. 注册中心（Eureka，Nacos）

### 1.1Eureka

* 问题1：服务1如何得知服务2的实例地址（服务1 调用 服务2）？

  获取地址信息的流程如下：

  * **服务注册：**服务2的服务实例启动后，会将自己的信息注册到 Eureka服务端 。这就叫做服务注册。
  * **服务端保存映射关系：**Eureka服务端保存`服务名称`到`服务实例地址列表`的映射关系
  * **服务发现：**服务1根据服务名称，拉取实力地址列表。这就叫做服务发现

* 问题2：服务1如何从多个服务2的实例中选择一个具体的实例？

  * 服务1 从实力列表中，利用负载均衡算法，选择一个实例地址
  * 向该实例地址发起远程调用

* 服务1 如何得知某个 服务2实例是否仍然健康？是不是已经宕机？

  * 服务2（被调用方）会每隔一段时间（默认30秒），向Eureka服务端报告自己的状态，**称为心跳**。
  * 超过一定时间没有发送心跳时，Eureka服务端会认为微服务实例故障，将该实例从服务列表中删除。
  * 服务1（调用方）拉取服务时，就能将故障实例排除了。

* **注意，一个微服务，既可以是服务提供者，也可以是服务调用者，因此Eureka将服务注册，服务发现等功能统一封装到了Eurkea-client端**。

（1）具体实现（orderservice在8080，userservice有两个实例，分别部署在8081和8082）：

   1. 服务发现、服务注册都封装在eureka-client依赖，因此两个service都要引入`eureka-client`依赖。

   2. **服务注册：**user service是服务提供者，我们需要在这个服务里面修改yaml，添加服务名称、eureka地址。

      eureka.client.service-url 指定服务端的注册地址，这个是客户端使用的，告诉客户端服务的地址，它可以指定多个，有一个默认的defaultZone。

			3.  **服务发现：**现在将orderService的逻辑修改，向eureka-server拉取user service的信息，实现服务发现。

       * 引入依赖，指定注册地址（就是前面的那个）

       * 在orderservice 的主程序中，给restTemplate这个bean添加一个@LoadBanlanced注解。（这样就实现了负载均衡）。

       * 修改orderservice服务中的orderservice类中的queryOrderById方法。修改访问的url路径，用服务名代替ip、端口：

         ```java
         public Order queryOrderById(Long orderId) {
                 // 1.查询订单
                 Order order = orderMapper.findById(orderId);
                 String url = "http://userservice/user/" + order.getUserId();
                 User user = restTemplate.getForObject(url, User.class);
                 order.setUser(user);
                 // 4.返回
                 return order;
             }
         ```

       spring 会自动帮助我们从eureka-server端，根据userservice这个服务名称，获取实力列表，而后完成负载均衡。

			4.  **Ribbon负载均衡**

       springCloud其实是使用了一个名为ribbon的组件，来实现负载均衡。

       ![image-20240323202022271](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240323202022271.png)

       springCloudRibbon的底层采用了一个拦截器，拦截了restTemplate发出的请求，对地址做出了修改。

       ![image-20240323202201453](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240323202201453.png)

       **基本流程如下：**

        * 拦截rest Template请求`http://userservice/user/1`
        * ribbonLoadBanlancerClient会从请求url中获取服务名称，即user service
        * DynamicServerListLoadBalander根据user service到eureka拉取服务列表。
        * eureka返回列表，user service拢共两个实例，那么就返回这两个实例的地址。
        * IRule利用内置负载均衡规则，从列表中挑选一个服务地址（假设是localhost:8081）。
        * RibbonLoadBalancer修改请求地址，用localhost:8081替代user service，得到http://localhost:8081/user/1，发起真实请求

       **Ribbon默认是采用懒加载，即第一次访问时才会去创建loadBalanceClient，请求时间会很长。**而饥饿加载、会在项目启动时创建，降低第一次访问的耗时。通过下面的配置开启饥饿加载：

       ```java
       ribbon:
         eager-load:
           enabled: true # 开启饥饿加载
           clients:
             - userservice # 指定饥饿加载的服务名称
             - xxxxservice # 如果需要指定多个，需要这么写
       ```

### 1.2 Nacos

1. **与eureka的联系：**springCloudAlibaba也遵循spring Cloud中定义的服务注册、服务发现规范。因此使用Nacos和使用Eureka对于微服务来说，并没有太大区别。nacos跟eureka的主要区别在于（1）依赖不同（2）服务地址不同

2. **前期准备工作：**安装nacos，**nacos默认运行在8848端口**。进入安装目录下的bin目录，执行`startup.cmd -m standalone`即可开启nacos服务。

3. **引入依赖：**

   * 在父工程的pom文件中引入spring Cloud Alibaba的依赖。
   * 在两个服务userservice和orderService中的pom文件中引入nacos-discovery依赖。

4. **配置nacos地址：**在userservice和orderservice的application.yml中添加nacos地址如下：

   ```yaml
   spring:
     cloud:
       nacos:
         server-addr: localhost:8848
   ```

5. **重启微服务后，登录nacos管理页面，可以看到微服务信息。**

6. **服务分级存储模型：**

   * 一个服务可以有多个实例，假如这些实例分布于全国各地的不同机房，nacos就会将同一机房内的实例划分为一个**集群**。

   * 也就是说，一个服务可以有多个集群，这些集群下面就是该服务的多个实例。
   * 微服务互相访问时，应该尽可能访问同集群实例，因为本地访问速度更快。当本集群内不可用时，才会访问其他集群。

7. **给user-service配置集群：**

   ```java
   spring:
     cloud:
       nacos:
         server-addr: localhost:8848
         discovery:
           cluster-name: HZ # 集群名称
   ```

8.  **同集群优先的负载均衡：**默认的ZoneAvoidanceRule并不能

## 2. 消息队列MQ（RabbitMQ 、SpringAMQP）

* **同步调用的问题（微服务间基于Feign的调用就属于同步方式，存在一些问题）：**

  * （1）耦合导致改动很麻烦：每次加入新的需求，都要修改原来的代码；
  * （2）性能与吞吐量的下降：调用者需要等待服务提供者相应，如果调用链过长则响应时间等于调用的时间之和；
  * （3）产生了严重的资源浪费：调用链中的每个服务在等待响应过程中，不能释放请求占用的资源，高并发场景下会极度浪费系统资源；
  * （4）级联失败：如果服务提供者出现问题，所有调用方都会跟着出问题；

* **异步调用方案：解决了上述的同步调用的问题。引入事件代理Broker，用户服务向broker发布事件，由broker进行通知。**

  * 另一个优势是，可以进行**流量削峰**，broker可以起到一个缓冲的作用，降低服务的压力。
  * 总结来说，异步通信的优点在于（1）耦合度低（2）吞吐量提升（3）故障隔离（4）流量削峰；异步通信的缺点在于**（1）依赖于Broker的可靠性，安全性以及吞吐能力（2）架构复杂了，业务没有明显的流程线，不好追踪管理**。

* **各个MQ消息队列之间的对比：**

  ![image-20240326151804916](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240326151804916.png)

* RabbitMQ的broker工作流程：

  ![image-20240326152825654](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240326152825654.png)

* **Rabbit中的几个概念：**

  * channel：操作MQ的工具
  * exchange：路有消息到队列中
  * queue：缓存消息
  * virtual host：虚拟主机，是对queue、exchange等资源的逻辑分组。

* **RabbitMQ常见消息模型：**

  ![image-20240326153454835](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240326153454835.png)



### 2.1 springAMQP

* AMQP是应用间消息通信的一种协议，与语言和平台无关。

* springAMQP是基于AMQP协议定义的一套API规范，提供了模板来发送和接收消息。包含两部分，其中spring-amqp是基础抽象，spring-rabbit是底层的默认实现。

* **利用SpringAMQP实现helloworld中的基础消息队列功能**

  * 流程如下：

    * 1.在父工程中引入spring-ampq的依赖。
    * 2.在publisher服务中利用**RabbitTemplate**发送消息到simple.queue这个队列。
    * 3.在consumer服务中编写消费逻辑，绑定simple.queue这个队列。

  * 首先，配置publisher和consumer的application.yaml:

    ```java
    # 1.1.设置连接参数，分别是：主机名、端口号、用户名、密码、vhost
    spring:
      rabbitmq:
        host: 127.0.0.1
        port: 5672
        username: root
        password: root
        virtual-host: /
    ```

  * 然后编写测试类springAmqpTest用于实现消息发送:

    ```java
    @Autowired
    private RabbitTemplate rabbitTemplate
    @Test
    public static void testSendMessage2SimpleQueue () {
    	String queueName = "simple.queue";
        String message = "hello";
        rabbitTemplate.convertAndSend (queueName,message);
    }
    ```

  * 在consumer中新建一个类,来实现消息的接收：

    ```java
    @Component
    public class SpringRabbitListener {
        @RabbitListener(queues = "simple.queue")
        public void listenSimpleQueueMessage(String msg) throws InterruptedException {
            System.out.println("spring消费者接收到消息：" + meg);
        }
    }
    ```

    **总结，SpringAMQP如何接收消息：**

    * 引入amqp的starter依赖
    * 配置RabbitMQ地址
    * 定义类，添加@Component注解
    * 类中声明方法 ，添加@RabbitListener注解，方法参数就是消息。

    注意，消息一旦消费就会从队列中删除，rabbitMQ没有消息回溯功能。

* **SpringAMQP消息预取机制**

  * 预取机制就是，当消息进入到消息队列的时候，消费者不管自己的消费能力有多少，轮流预取队列中的消息，随后进行消费，这会导致低消费能力的consumer也有可能处理很多的消息。
  * **解决办法：**修改application.yaml文件，设置`spring-rabbitmq-listener-simple-prefetch`属性，可以控制预取消息的上限。

* **发布订阅模型：**发布订阅模式与之前的区别就在于允许将同一消息发送给多个消费者，实现方式是加入了exchange（交换机）（也就是说，同一个消息可以被多个消费者消费）

  ![image-20240326170444055](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240326170444055.png)

* **FanoutExchange**

  * 交换机的作用是什么？
    * 接收publisher发送的消息
    * 将消息按照规则路由到与之绑定的队列（fanout是广播，会路由到所有队列）
    * 不能缓存消息，路由失败，消息会丢失
    * FanoutExchange会将消息路由到每个绑定的队列
  * 声明队列、交换机、绑定关系的Bean是什么？
    * Queue
    * FanoutExchange
    * Binding

* **DirectExhange:** 其会将接收到的消息根据路由规则路由到指定的Queue，因此称为路由模式（router）

  * 每一个Queue都与Exchange设置一个BindingKey
  * 发布者发布消息时，指定消息的RoutingKey
  * Exchange将消息路由到BindingKey与消息RoutingKey一致的队列

