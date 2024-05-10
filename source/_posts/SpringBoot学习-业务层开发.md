---
title: SpringBoot学习-业务层开发
date: 2023-04-09 13:44:54
tags:
---

#### service（业务）层接口与dao（数据）层接口的区别：

* 业务层接口关注的是业务名称，数据层接口关注的是数据库相关的操作

#### 业务层开发（常规方式）：

1. 自定义业务接口：

```java
public interface UserService {
    Boolean save(User user);
    Boolean update(User user);
    Boolean delete(Integer id);
    User getById(Integer id);
    List<User> getAll();
}
```

2. 定义业务层实现类（要记得在实现类上面添加@Service注解）：

   ```java
   @Service
   public class UserServiceImpl implements UserService {
       @Autowired
       UserMapper userMapper;
       @Override
       public Boolean save(User user) {
           return userMapper.insert(user) > 0;
       }
   
       @Override
       public Boolean update(User user) {
           return userMapper.updateById(user) > 0;
       }
   
       @Override
       public Boolean delete(Integer id) {
           return userMapper.deleteById(id) > 0;
       }
   
       @Override
       public User getById(Integer id) {
           return userMapper.selectById(id);
       }
   
       @Override
       public List<User> getAll() {
           return userMapper.selectList(null);
       }
   }
   ```

#### 业务层快速开发（Mybatis-Plus提供）：

1. 定义业务层接口，该接口继承IService接口，并指定泛型：

```java
public interface IUserService extends IService<User> {
}
```

2. 定义业务层实现类，需要继承MybatisPlus提供的ServiceImpl类，并指定泛型`<UserMapper,User>`,同时实现上文定义的`IUserService`接口:

```java
@Service
public class IUserServiceImpl extends ServiceImpl<UserMapper, User> implements IUserService {
}
```

3. 如果其提供的功能不能满足开发需求的话，可以在通用类基础上做功能重载或功能追加（注意重载时不要覆盖原始操作，避免原始提供的功能丢失）。



