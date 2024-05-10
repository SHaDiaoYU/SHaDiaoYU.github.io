---
title: java反射机制学习
date: 2023-03-26 21:01:27
tags:
---

### java 反射(reflection)机制学习

1. 反射知识重点：反射机制、Class类、类加载、反射获取类的结构信息（Class \ Field \ Method \ Constructor \ 访问属性 \ 访问方法）。
2. 反射知识扩展：反射相关类、反射调用性能优化、Class类常用方法。

*以需求来引入反射：*

根据配置文件re.properties指定信息，创建Cat对象并调用方法hi():

```properties
classfullpath = com.java.cat

method = hi
```

* 这样的需求在学习框架时特别多，即通过外部文件配置，在不通过修改源码的情况下控制程序，也符合设计模式的OCP原则（开闭原则：不修改源码，扩容功能）

* 传统的方式： 先创建对象，然后再调用其中的方法（发现这种方式来实现上述需求很困难）；

*引入反射机制，使用反射机制解决：*

1. 加载类，返回Class类型的对象（需要先用IO流读到classFullPath）:

   `Class cls = Class.forName(classfullpath);`

2. 通过cls得到加载的类 com.java.Cat 的实例：

   `Object o = cls.newInstance();`

3. 通过 cls 得到加载的类 com.java.Cat 的methodName "Hi"的方法对象（需要先用IO流读到methodName）。即：在反射中，可以把方法看作对象。 

   `Method method1 = cls.getMethod(methodName);`

4. 通过method1 调用方法： 即通过方法对象来实现调用方法

   `method1.invoke(o);`

5. 获取字段，同样地，传入的参数为字段名（注意：getField方法不能得到私有的属性）

   `Field nameField = cls.getField("age");`

6. 获取无参构造器, ()中可以指定构造器参数类型，返回无参构造器

   `Constructor constructor = cls.getConstructor();`

7. 获取有参构造器，()中传入的是有参构造器参数列表的类（.class）

   `Constructor constructor2 = cls.getConstructor(String.class);`

这样操作，就可以仅通过操作配置文件，来调用不同的方法了。

---------------------------反射机制--------------------------------

##### java Reflection反射机制:

* 反射机制允许程序在执行期间借助于`ReflectionAPI` 取得任何类的内部信息（比如成员变量、构造器、成员方法等等），并能操作对象的属性及方法。反射在设计模式和框架底层都会用到。
* 加载完类之后，堆中产生Class类型的对象（一个类只有一个Class类型的对象），这个对象包含了类的完整结构信息。通过这个对象得到类的结构。这个Class对象就像一面镜子，透过这个镜子看到类的结构，所以被形象的称之为反射。

##### Java反射机制可以做的事情：

* 在运行时判断任意一个对象所属的类
* 在运行时构造任意一个类的对象
* 在运行时得到任意一个类所具有的成员变量和方法
* 在运行时调用任意一个对象的成员变量和方法
* 生成动态代理

##### Java反射相关的类：

* java.lang.Class: 代表一个类，Class对象表示某个类加载在堆中的对象
* java.lang.reflect.Method: 代表类的方法，Method对象表示某个类的方法
* java.lang.reflect.Field: 代表类的成员变量
* java.lang.reflect.Constructor: 代表类的构造方法

##### 反射的优缺点：

* 优点： 可以动态的创建和使用对象（是框架的底层和核心），使用灵活，没有反射机制，框架技术就失去底层支撑。
* 缺点： 使用反射基本是解释执行，对执行速度有影响。

##### 关于Class类：

1. Class也是类，因此也会继承Object类
2. Class类对象不是new出来的，而是**由系统创建**的
3. 对于某个类的Class类对象，在内存中只有一份，因为**类只加载一次**（重点）
4. 每个类的实例都会记得自己是由哪个Class实例所生成
5. 通过Class可以完整地得到一个类的完整结构，通过一系列API
6. Class对象是存放在**堆**内的
7. 类的字节码二进制数据(跟.class字节码文件是不同的)，是放在方法区的，有的地方称为类的元数据（包括方法代码、变量名、方法名、访问权限等）

![](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/QQ%E6%88%AA%E5%9B%BE20230327194724.png)

##### 获取java对象的方式：

1. 在代码阶段/编译阶段： 使用`Class.forName()`获取

   ​	**前提**：已知一个类的全类名，且该类在类路径下，可通过Class类的静态方法`forName()`获取，可能抛出ClassNotFoundException;

   ​	**实例**：`Class cls1 = Class.forName("java.lang.Cat");`

   ​	**应用场景**：多用于配置文件，读取全类名，加载类；

2. 在Class类阶段(加载阶段)：使用`类.class`获取

   ​	**前提**：已知具体的类，通过类的class获取，该方式最为安全可靠，程序性能最高；

   ​	**实例**：`Class cls2 = Cat.class;`

   ​	**应用场景：** 多用于参数传递，比如通过反射得到对应构造器对象； 

3. 在运行阶段(此时已经有对象被创建出来了)：使用`对象.getClass()`获取

4. 通过类加载器ClassLoader获取：

   ```java
   // 首先获取到类加载器
   ClassLoader classLoader = cat.getClass().getClassLoader();
   // 通过类加载器得到class对象
   Class cls4 = classLoader.loadClass("com.java.Cat");
   System.out.println(cls4);
   ```

5. 基本数据类型（int,char,boolean,float,double,byte,long,short）按照如下方式得到Class类对象：

   ​	`Class cls = 基本数据类型.class`

6. 基本数据类型对应的包装类，可以通过`.TYPE`得到Class类对象

   ​	**实例：**`Class cls = 包装类.TYPE`

##### 哪些类型有Class对象：

* 外部类、成员内部类、静态内部类、局部内部类、匿名内部类；
* interface
* 数组
* enum枚举
* annotation注解
* 基本数据类型
* void

##### 类加载机制

反射机制是java实现动态语言的关键，也就是通过反射实现类动态加载。

* **静态加载**：编译时加载相关的类，如果没有则报错，依赖性过强
* **动态加载**：运行时加载需要的类，如果运行时不用该类（即使不存在该类），则不报错，降低了依赖性

**类加载的时机**：

* 当创建对象时 <==== 属于静态加载；
* 当子类被加载时（子类被加载则父类也需要加载）<==== 属于静态加载；
* 调用类中的静态成员时<==== 属于静态加载；
* 通过反射<==== 属于动态加载；

