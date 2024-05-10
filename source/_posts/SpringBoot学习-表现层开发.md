---
title: SpringBoot学习-表现层开发
date: 2023-04-10 14:07:08
tags:
---

#### SpringBoot表现层消息一致性处理（R对象）：

在开发中，后端返回给前端的数据格式，有时是单数据对象，有时是json对象，有时又会是json数组等等······，没有固定的格式，这样会给前端的开发带来不必要的麻烦，因此引入了消息一致性处理，统一消息格式来解决此类问题。

**解决方式**:

* 设计表现层返回结果的模型类(R)，用于后端与前端进行数据格式的统一，也称为**前后端数据协议**

**方法内容：**

1. 第一步，创建模型类R, 添加有参/无参构造器（格式只要前端与后端协商好就可以，不一定非要用下面的格式）；

```java
@Data
public class R {
    private Boolean flag;
    private Object data;

    public R(){}

    public R(Boolean flag){
        this.flag = flag;
    }
    
    public R(Boolean flag,Object data){
        this.flag = flag;
        this.data = data;
    }
}
```

2. 第二步， 在Controller中使用R对象，对返回的结果进行封装；

```java
@RestController
@RequestMapping("/users")
public class UserController2 {
    @Autowired
    private IUserService service;

    @GetMapping
    public R getAllUser() {
        return new R(true,service.list());
    }

    @PostMapping
    public R InsertUser(User user){
        return new R(service.save(user));
    }

    @DeleteMapping("{id}")
    public R DeleteUser(@PathVariable("id") Integer id){
        return new R(service.removeById(id));
    }
    
    .......
    
}
```

#### 异常消息处理

如果程序发生异常，后台返回500、404等错误信息，这些错误信息也应能与正确信息的数据格式保持一致：

**解决方式：**

* 使用springMVC提供的异常处理器，通过该异常处理器来将所有的异常给处理掉

**方法内容：**

1. 第一步，在controller /utils 包下创建一个新的类（这里我们命名为ProjectExceptionAdvice）;

```java
//作为springMVC的异常处理器
@ControllerAdvice
public class ProjectExceptionAdvice {

    //这样定义的方法就会拦截所有的异常信息
    @ExceptionHandler
    public R doException(Exception ex){
        //一定要打印异常信息，不然在控制台什么都看不到
        ex.printStackTrace();
        return new R("服务器故障，请稍后再试！");
    }
}
```

注意，配置异常处理器需要添加`@ControllerAdvice`注解，在拦截异常方法上，添加`@ExceptionHandler`注解，可以指定处理异常的类型，形如：`@ExceptionHandler(IOException.class)`

2. 因为我们需要返回错误信息，所以需要为R类添加一个字段message用以返回错误信息，同时需要修改或者添加构造器来实现我们的目的：

```java
@Data
public class R {
    private Boolean flag;
    private Object data;
    private String message;

    public R(){}

    public R(Boolean flag){
        this.flag = flag;
    }

    public R(Boolean flag,Object data){
        this.flag = flag;
        this.data = data;
    }

    public R(String message){
        this.flag = false;
        this.message = message;
    }

    public R(Boolean flag,String message){
        this.flag = flag;
        this.message = message;
    }
```

3. 改写我们的controller,使其返回错误信息。在前端也使用`this.$message.error(resp.data.msg)`进行错误信息的展示（前端部分不做展示）：

```java
@RestController
@RequestMapping("/users")
public class UserController2 {
    @Autowired
    private IUserService service;

    @PostMapping
    public R InsertUser(@RequestBody User user){
        boolean flag = service.save(user);
        return new R(flag,flag ? "添加成功！" : "添加失败！"); // 使用三目运算来返回结果
    }
    ........
        
}
```





