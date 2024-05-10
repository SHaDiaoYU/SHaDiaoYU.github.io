---
title: SpringBoot学习-缓存相关内容
date: 2023-06-06 15:26:57
tags:
---

#### 1.SpringBoot缓存技术

* springboot提供了缓存技术，方便缓存使用。

  * 导入坐标：

  ```xml
  <dependency>
  	<groupId>org.springframework.boot</groupId>
  	<artifactId>spring-boot-starter-cache</artifactId>
  </dependency>
  ```

  * 在启动类中添加`@EnableCaching`注解，开启缓存功能。

  * 在涉及查询的方法上添加`@Cachable(value=缓存空间名称,key=#变量名)`，表明该方法返回的结果会放入该名称的缓存空间下。示例如下：

    ```java
    @Cachable(value="cacheSpace",key="#id")
    public Book getById(Integer id) {
        return bookMapper.selectById(id);
    }
    ```

* springboot提供的缓存技术除了提供默认的缓存方案，还可以与其他缓存技术进行整合，统一接口，方便缓存技术的开发与管理。

  * 这些缓存技术包括 simple(默认)、Generic、Jcache、**Ehcache**、Hazelcast、Infinispan、Couchbase、**Redis**、Caffenine。
  * 也有未被springboot整合的缓存技术：常用的是**memcached**。

#### 2.缓存使用案例--手机验证码

* 需求：

  * 输入手机号获取验证码，组织文档以短信形式发送给用户（页面模拟）
  * 输入手机号和验证码验证结果。

* 需求分析：

  * 提供controller，传入手机号，业务层通过手机号计算出独有的6位验证码数据，存入缓存后返回此数据；
  * 提供controller，传入手机号与验证码，业务层通过手机号从缓存中读取验证码并与输入验证码进行比对，返回比对结果；

* 具体实现示例：

  * 第一步：创建bean，名为 SMSCode:

    ```java
    @Data
    public class SMSCode {
        private String tele; // 电话号码
        private String code; // 验证码
    }
    ```

  * 第二步，创建用于生成验证码的工具类CodeUtils:

    ```java
    @Component
    public class CodeUtils {
    
        // 用于补0的数组
        private String[] patch = {"000000","00000","0000","000","00","0",""};
    
        // 根据电话号码生成验证码
        public String generator(String tele){
            int hash = tele.hashCode();
            // 设置加密码
            int encryption = 20206666;
            // 与hashcode进行异或以完成加密
            long result = hash ^ encryption;
            // 为使验证码不唯一，获取系统当前时间，并与其再次进行异或
            long nowTime = System.currentTimeMillis();
            result = nowTime ^ result;
            // 获取后六位即可
            long code = result % 1000000;
            // 解决code值可能为负的问题
            code = code < 0 ? -code:code;
            String codeStr = code + "";
            int length = codeStr.length();
            return  patch[length] + codeStr;
        }
    
        @Cacheable(value = "smsCode", key = "#tele")
        public String getCodeFromCache(String tele) {
            return null;
        }
    }
    ```

    注意，这里把getCodeFromCache方法放在这个工具类之中的原因，是我们需要使用springboot容器统一管理，如果放在service下，@Cacheable注解将会失效，原因是其未被加载至springboot容器内。

  * 第三步，新建业务层接口SMSCodeService及其实现类SMSCodeServiceImpl

    ```java
    public interface SMSCodeService {
        // 向手机发送验证码
        public String sendCodeToSMS(String tele);
        // 校验验证码
        public boolean check(SMSCode code);
    }
    ```

    ```java
    @Service
    public class SMSCodeServiceImpl implements SMSCodeService {
        @Autowired
        private CodeUtils codeUtils;
    
        @Override
        // 只用往缓存内存放验证码以供校验使用，使用下列注解
        @CachePut(value = "smsCode",key = "#tele")
        public String sendCodeToSMS(String tele) {
            return codeUtils.generator(tele);
        }
    
        @Override
        public boolean check(SMSCode smsCode) {
            String code = smsCode.getCode();
            String cacheCode = codeUtils.getCodeFromCache(smsCode.getTele());
            return code.equals(cacheCode);
           // return false;
        }
    }
    ```

  * 第四步，编写控制层SMSCodeController类用以测试：

  ```java
  @RestController
  @RequestMapping("/sms")
  public class SMSCodeController {
      @Autowired
      private SMSCodeService service;
  
      @GetMapping
      public String getCode(String tele){
          String code = service.sendCodeToSMS(tele);
          return code;
      }
      @PostMapping
      public boolean checkCode(SMSCode smsCode){
          return service.check(smsCode);
      }
  }
  ```

  完成这些步骤后，在postman中进行测试即可。

#### 3. jetcache缓存方案

jetcache缓存方案包括 远程缓存方案 和 本地缓存方案 两种。

**（1）jetcache远程缓存方案：**

* 首先，在pom文件中添加jetcache的坐标。然后，在application.yml中配置jetcache远程方案默认类型为redis，设置主机地址，端口（注意，poolconfig这一项必须配置，至少配置一条，否则初始化的时候会报错。）:

```xml
<dependency>
	<groupId>com.alicp.jetcache</groupId>
	<artifactId>jetcache-starter-redis</artifactId>
	<version>2.6.2</version>
</dependency>
```

```yaml
jetcache:
  remote:
    default:
      type: redis
      host: 192.168.153.137
      port: 6379
      poolconfig:
        maxTotal: 50
```

* 第二步，在启动类中添加`@EnableCreateCacheAnnotation`注解，这个注解是jetcache启动缓存的主开关。
* 定义Cache<K,V>对象，并添加注解@CreateCache，告诉jetcache这是用于进行缓存的对象。name属性定义缓存空间名称，expire属性定义存活时间（默认单位为秒）

```java
	@CreateCache(name = "smsCache",expire = 3600)
    private Cache<String,String > jetCache;
```

* 使用put和get方法即可进行缓存存取值：

```java
	 @Override
    public String sendCodeToSMS(String tele) {
        String code = codeUtils.generator(tele);
        jetCache.put(tele,code);
        return code;
    }
    @Override
    public boolean check(SMSCode smsCode) {
        String code = smsCode.getCode();
        String cacheCode = jetCache.get(smsCode.getTele());
          return code.equals(cacheCode);
    }
```

![image-20230609143218491](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230609143218491.png)

* 在redis客户端进行查看，可以看到新添加的缓存。可以看到我们设置的name跟key连到一块了。所以在设置name时最好在后面加上分隔符，比如“smsCode_”，这样可以更好地做区分。

**（2）jetcache本地缓存方案**

* 首先，添加jetcache本地缓存配置：

```yaml
jetcache:
  local:
    default:
      type: linkedhashmap
      keyConverter: fastjson
```

* 然后，设置缓存类型为local（其中有三种类型可选，BOTH代表本地和远程同时缓存；LOCAL代表使用本地缓存；REMOTE代表使用远程缓存）

![image-20230609154601714](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230609154601714.png)

**（3）jetcache方法缓存**

这里的方法缓存是指在方法上添加注解使其具有缓存的功能，具体流程如下：

* 首先，在启动类上添加`@EnableMehodCache`注解，并配置路径，使用缓存的类必须存在该路径内。同时，该注解必须与@EnableCacheAnnotation 注解配合使用：

  ```java
  @SpringBootApplication
  // jetcache启用缓存的主开关
  @EnableCreateCacheAnnotation
  // 启用方法缓存
  @EnableMethodCache(basePackages = "com.jiawei.boot")
  public class BootApplication {
      public static void main(String[] args) {
          SpringApplication.run(BootApplication.class, args);
      }
  }
  ```

* 第二步，在方法上添加`@Cached`注解，指明缓存空间名称，关键字，存放时间：

  ```java 
  	@Cached(name="book_",key="#id",expire = 3600)
  	public Book getById(Integer id) {
          return bookMapper.selectById(id);
      }
  ```

* 这之后运行会报一个空指针的异常，这个异常是由于默认使用的方案，而我们没有配置keyConverter引起的。添加该项配置即可：

  ```yaml
  jetcache:
    remote:
      default:
        type: redis
        host: 192.168.153.137
        port: 6379
        keyConverter: fastjson
        poolconfig:
          maxTotal: 50
  ```

* 再然后，会爆出一个未序列化的异常，这个异常是由于redis没有存储对象的功能，所以我们需要进行序列化对象进行存储。添加Book对象的序列化配置过程如下：

  ```java
  public class Book implements Serializable{
      ...
  }
  ```

  配置中添加：

  ```yaml
  jetcache:
    remote:
      default:
        type: redis
        host: 192.168.153.137
        port: 6379
        keyConverter: fastjson
        valueEncode: java
        valueDecode: java
        poolconfig:
          maxTotal: 50
  ```

* 至此问题解决，但是也存在着当数据库进行更新以及删除操作时，缓存的数据需要进行同步的问题，我们为更新和删除操作添加注解，每当进行更新或删除操作时就更新缓存内容：

```java
// 更新方法
@CacheUpdate(name="book_",key="#book.id",value="#book")
public boolean update(Book book) {
    return bookMapper.update(book)
}
// 删除方法
@CacheInvalidate(name="book_",key="#id")
public boolean deleteBook(Integer id) {
    return bookMapper.deleteById(id)
}
```

* 设定缓存刷新（单位为秒）使用`@CacheRefresh（refresh = 5）`注解，此操作请谨慎使用。

#### 4. J2cache缓存框架

**（1）整合ehcache:**

* 第一步，导入j2cache、ehcache依赖，注意j2cache需要导入两处依赖：

  ```xml
  <dependency>
  	<groupId>net.oschina.j2cache</groupId>
  	<artifactId>j2cache-core</artifactId>
  	<version>2.6.0-release</version>
  </dependency>
  <dependency>
  	<groupId>net.oschina.j2cache</groupId>
  	<artifactId>j2cache-spring-boot2-starter</artifactId>
  	<version>2.7.5-release</version>
  </dependency>
  <dependency>
  	<groupId>net.sf.ehcache</groupId>
  	<artifactId>ehcache</artifactId>
  </dependency>
  ```

* 第二步，application.yml中添加配置，指定j2cache配置文件路径：

  ```yaml
  j2cache:
    config-location: j2cache.properties
  ```

  添加ehcache的配置文件ehcache.xml：

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:noNamespaceSchemaLocation="http://ehcache.org/ehcache.xsd"
           updateCheck="false">
      <defaultCache
              eternal="false"
              maxElementsInMemory="1000"
              overflowToDisk="false"
              diskPersistent="false"
              timeToIdleSeconds="60"
              timeToLiveSeconds="60"
              memoryStoreEvictionPolicy="LRU"/>
  
  </ehcache>
  ```

  创建j2cache.properties配置文件，这里我们配置一级缓存(ehcache)和二级缓存(redis)，配置内容如下：

  ```properties
  # 1级缓存
  j2cache.L1.provider_class = ehcache
  ehcache.configXml = ehcache.xml
  
  # 2级缓存
  j2cache.L2.provider_class =net.oschina.j2cache.cache.support.redis.SpringRedisProvider
  j2cache.L2.config_section = redis
  redis.hosts = 192.168.153.127:6379
  
  # 1级缓存中的数据如何到达2及缓存
  j2cache.broadcast = net.oschina.j2cache.cache.support.redis.SpringRedisPubSubPolicy
  ```

  SpringRedisProvider的位置，（SpringRedisPubSubPolicy在同一级）：

  <img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20230613144723962.png" alt="image-20230613144723962" style="zoom:67%;" />

* 使用：创建CacheChannel对象，进行set和get：

  ```java
  @Service
  public class SMSCodeServiceImpl implements SMSCodeService {
      @Autowired
      private CodeUtils codeUtils;
      @Autowired
      private CacheChannel cacheChannel;
      @Override
      public String sendCodeToSMS(String tele) {
          String code = codeUtils.generator(tele);
          cacheChannel.set("sms",tele,code);
          //jetCache.put(tele,code);
          return code;
      }
      @Override
      public boolean check(SMSCode smsCode) {
          String code = smsCode.getCode();
          String cacheCode = cacheChannel.get("sms",smsCode.getTele()).asString();
            return code.equals(cacheCode);
      }
  }
  ```

  









