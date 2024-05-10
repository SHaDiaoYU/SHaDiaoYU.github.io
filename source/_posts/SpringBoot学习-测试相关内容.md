---
title: SpringBoot学习-测试相关内容
date: 2023-05-29 14:21:21
tags:
---

#### 1. 加载测试专用属性

假定，测试人员在测试的时候想要为当前的测试用例临时添加一些属性或者更改一些属性配置的值，应该怎么去做？

* properties属性可以为当前测试用例添加临时属性配置值：

```java
@SpringBootTest(properties = {"test.properties = testValue1"})
public class PropertiesAndArgsTest {
    @Value("${test.properties}")
    public String msg;
    ......
}
```

* args属性可以为当前测试用例添加临时的命令行参数：

```java
@SpringBootTest(args = {"--test.properties=testValue1"})
public class PropertiesAndArgsTest {
    @Value("${test.properties}")
    public String msg;
    ......
}
```

那么，如果properties和args配置了同一条规则，哪一个会生效呢？或者说哪一个优先级更高？

```java
@SpringBootTest(properties = {"test.properties = testValue1"},args = {"--test.properties=testValue2"})
public class PropertiesAndArgsTest {
    @Value("${test.properties}")
    public String msg;
    ......
}
```

输出msg，结果为testValue2，由此可见args的级别优先级更高。

很好理解，args是相当于为命令行添加参数，而命令行参数级别比配置文件级别更高，所以args的级别优先级更高。

**总结：**加载测试临时属性适用于小范围测试环境。

新的需求：如果测试人员想要在测试时添加新的bean辅助进行测试，又该怎么办？

#### 2.加载测试专用配置

* 首先，在test目录下添加测试会用到的bean:

```java
@Configuration
public class MsgConfig {
    @Bean
    public String msg(){
        return "this is bean msg.";
    }
}
```

* 使用**@Import({MsgConfig.class})**注解，导入bean，即可使用自动装配:

```java
@SpringBootTest
@Import({MsgConfig.class})
public class ConfigurationTest {
    @Autowired
    public String msg;

    @Test
    void testConfiguration(){
        System.out.println(msg);
    }
}
```

#### 3. 进行表现层测试

以往进行表现层测试时，我们使用postman来进行接口的测试，现在要求在做测试的时候所有功能，所有接口都必须测试一遍，那么该如何进行呢？

* web环境模拟

在测试类中启动web环境，使用`@SpringBootTest(webEnvironment = ...)`来进行设置：

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
public class WebTest {
    @Test
    void webTest(){
        ......
    }
}
```

webEnvironment有四种枚举值：**MOCK，DEFINED_PORT（定义好的端口，从配置文件中读取）,RANDOM_PORT（随机端口）,NONE（不启用web环境）**。

* 虚拟请求测试：

  （1）首先，使用`@AutoConfigMockMVC`注解，开启虚拟MVC调用。
  
  （2）第二步，使用自动装配方式，注入虚拟MVC调用对象：`MockMvc mvc`；
  
  （3）第三步，创建虚拟请求，设置访问路径：
  
  ```java
   MockHttpServletRequestBuilder mockHttpServletRequestBuilder = MockMvcRequestBuilders.get("/users");
  ```
  
  （4）第四步，执行请求：
  
  ```java
  ResultActions perform = mvc.perform(mockHttpServletRequestBuilder);
  ```
  
  总：
  
  ```java
  @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT)
  // 开启虚拟MVC调用
  @AutoConfigureMockMvc
  public class WebTest {
      // 注入虚拟MVC调用对象
      @Test
      void webTest(@Autowired MockMvc mvc) throws Exception {
          // 创建虚拟请求，设置访问路径
          MockHttpServletRequestBuilder mockHttpServletRequestBuilder = MockMvcRequestBuilders.get("/users");
          //执行请求
          ResultActions perform = mvc.perform(mockHttpServletRequestBuilder);
      }
  }
  ```

#### 4. 真实值与预期值匹配

在能够发送虚拟请求之后，我们需要验证当前返回的结果与预期值是否匹配。

* 第一种：虚拟请求状态匹配。我们根据虚拟请求的状态来进行匹配：

  ```java
  @Test
      void webTest(@Autowired MockMvc mvc) throws Exception {
          MockHttpServletRequestBuilder mockHttpServletRequestBuilder = MockMvcRequestBuilders.get("/user");
          ResultActions perform = mvc.perform(mockHttpServletRequestBuilder);
          // 设置预期值， 与真实值进行对比，成功则测试通过，失败则测试失败
          // 定义本次调用的预期值
          StatusResultMatchers status = MockMvcResultMatchers.status();
          // 预计本次调用成功时的状态：200
          ResultMatcher ok = status.isOk();
          // 添加预期值到本次调用过程中进行匹配
          perform.andExpect(ok);
      }
  ```

* 第二种：响应体匹配。我们根据虚拟请求的响应体来进行匹配。

```java
	@Test
    void testBody(@Autowired MockMvc mvc) throws Exception {
        // 创建虚拟请求，设置访问路径
        MockHttpServletRequestBuilder mockHttpServletRequestBuilder = 				MockMvcRequestBuilders.get("/users");
        //执行请求
        ResultActions perform = mvc.perform(mockHttpServletRequestBuilder);
        // 设置本次调用的预期值
        ContentResultMatchers content = MockMvcResultMatchers.content();
        ResultMatcher result = content.string("nihao");
        // 添加预期值到本次调用过程中进行匹配
        perform.andExpect(result);
    }
```

* 第三种：响应体（json格式）匹配。

```java
	@Test
    void testJsonBody(@Autowired MockMvc mvc) throws Exception {
        // 创建虚拟请求，设置访问路径
        MockHttpServletRequestBuilder mockHttpServletRequestBuilder = MockMvcRequestBuilders.get("/users");
        //执行请求
        ResultActions perform = mvc.perform(mockHttpServletRequestBuilder);

        //设置本次调用的预期值
        ContentResultMatchers content = MockMvcResultMatchers.content();
        ResultMatcher json = content.json("{\n    \"username\": \"zhangsan\",\n    \"password\": \"12344\"\n}");
        // 添加预期值到本次调用过程中进行匹配
        perform.andExpect(json);
    }
```

* 第四种：响应头匹配。根据虚拟请求的请求头（contentType）来进行匹配。

  ```java
  @Test
  void testHeader(@Autowired MockMvc mvc) throws Exception {
      // 创建虚拟请求，设置访问路径
      MockHttpServletRequestBuilder mockHttpServletRequestBuilder = MockMvcRequestBuilders.get("/users");
      //执行请求
      ResultActions perform = mvc.perform(mockHttpServletRequestBuilder);
  
      // 设置本次调用的 预期值
      HeaderResultMatchers header = MockMvcResultMatchers.header();
      ResultMatcher string = header.string("Content-Type", "application/json");
      // 添加预期值到本次调用过程中进行匹配
      perform.andExpect(string);
  
  }
  ```

上述匹配规则可以组合使用。

#### 5. 业务层测试事务回滚

我们期望，测试产生的结果数据不要存放在数据库中，只是关注它的过程数据就行。

* 方法：使用@Transaction注解即可：

  * 为用例添加事务，SpringBoot会对测试用例对应的事务提交操作并进行回滚：

  ```java
  @SpringBootTest
  @Transactional
  public class MapperTest {
      @Autowired
      UserService service;
  
      @Test
      void insertTest(){
          User user = new User();
          user.setUsername("zhangsan");
          user.setPassword("lisi");
          service.saveUser(user);
      }
  }
  ```

  * **注意：**如果想在测试用例中提交事务，可以通过**@RollBack**注解设置：

  ```java
  @SpringBootTest
  @Transactional
  @RollBack(false) // 如果为false,则不回滚，事务正常提交；如果为true，则回滚，事务不会被提交。
  public class MapperTest {
      @Autowired
      UserService service;
      @Test
      void insertTest(){
          ......
      }
  }
  ```

#### 6. 测试用例设置随机数据

在设置测试用例数据的时候，我们不能想上面那样直接用setter设置，而是使用springboot生成的随机数据完成。

使用方式：

```yaml
typecase:
 id: ${random.int}
 id2: ${random.int(10)} #1-10之间
 type: ${random.int(5-10)} #5-10
 name: 这是${random.value}
 uuid: ${random.uuid}
 publishTime: ${random.long}
```





