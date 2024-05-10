---
title: Cookie & Session
date: 2022-11-16 15:40:55
tags:
---

# Cookie与Session内容

## (一) cookie:

### （1）Cookie基本使用：

- Cookie：客户端会话技术，将数据保存到客户端，以后每次请求都携带Cookie数据进行访问

- Cookie基本使用：

  1. 创建Cookie对象，设置数据：

     ` Cookie cookie = new Cookie("key","value");`

  2. 发送Cookie到客户端(浏览器)，使用response对象：

     ` response.addCookie(cookie);`

  3. 获取客户端携带的所有cookie,使用request对象

     `Cookie[] cookies = request.getCookies();`

  4. 遍历数组，获取每一个Cookie对象：for循环  

  5. 使用Cookie对象方法获取数据

     ` cookie.getName();` 

     `cookie.getValue();`

### （2）Cookie原理：

1. Cookie的实现是基于HTTP协议的

   * 响应头（由服务端发出）：set-cookie

     ![](C:\Users\ShangJiaWei\AppData\Roaming\Typora\typora-user-images\image-20221116141255489.png)

   * 请求头（由客户端发出）：cookie

     ![image-20221116141406786](C:\Users\ShangJiaWei\AppData\Roaming\Typora\typora-user-images\image-20221116141406786.png)

### （3）Cookie的使用细节

1. Cookie存活时间：

   * 默认情况下，Cookie存储在浏览器内存中，浏览器关闭后，Cookie被销毁。
   * setMaxAge(int seconds)：设置Cookie存活时间
     1. 正数：将Cookie写入硬盘，持久化保存，到时间自动删除；
     2. 负数：默认值，Cookie在浏览器内存中，浏览器关闭，则Cookie自动销毁；
     3. 零：删除对应的Cookie；

2. Cookie存储中文：

   * Cookie不能直接存储中文；

   * 如果需要存储，则需要进行转码：URL转码

     ```java
     String value = "张三"；
     //URL 编码
     value = URLEncoder.encode(value,"UTF-8");
     Cookie cookie = new Cookie("username",value);
     ```

     ```java
     //URL解码
     value = URLDecoder.decode(value,"UTF-8");
     ```

     

## （二）Session：

### （1）Session的基本使用：

* Session：服务端会话跟踪技术，将数据保存到服务端。

* JavaEE提供HttpSession接口，来实现一次会话的多次请求间数据共享功能；

* 使用：

  1. 获取Session对象

     `HttpSession session = request.getSession();`

  2. Session对象功能

     * void setAttribute(String name,Object o):存储对象到session域中；
     * Object getAttribute(String name)：根据key，获取值；
     * void removeAttribute(String name)：根据key，删除该键值对；

### （2） Session的原理：

* Session是基于Cookie实现的，同一会话内进行`request.getSession()`后得到的是同一个对象；
* 同一会话内，首次获取Session对象时，生成一个SessionID，在响应时存入响应头`set-cookie:JSESSIONIDE=xxx`，之后的每一次请求的请求头内，携带sessionID(`cookie:JSESSIONID = xxx`)，服务端据此寻找是否有该ID的session对象，如果有就直接拿来用，以此保证拿到的session对象为同一个。

### （3） Session的使用细节：

* Session钝化、活化：

  * 服务器重启后，session中的数据是否还在？

    不会丢失，前提为服务器是正常启动停止的。

    * 钝化：在服务器正常关闭后，Tomcat会自动将Session数据写入硬盘的文件中；
    * 活化：再次启动服务器后，从文件中加载数据到Session中；

* Session销毁：

  * 默认情况下，无操作，30分钟自动销毁；

  * 调用Session对象的invalidate()方法；

    

## （三）小结

* Cookie和Session都是来完成一次会话内多次请求间**数据共享**的
* 区别：
  * 存储位置：Cookie是将数据存储在客户端，Session将数据存储在服务端；
  * 安全性：Cookie不安全，Session安全；
  * 数据大小：Cookie最大3KB，Session无大小限制；
  * 存储时间：Cookie可以长期存储，Session默认30分钟；
  * 服务器性能：Cookie不占用服务器资源，Session占用服务器资源；