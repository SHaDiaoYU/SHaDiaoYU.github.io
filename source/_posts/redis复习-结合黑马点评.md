

## Redis:

### Redis数据结构：

（1）5种基础数据结构：String（字符串）、List（列表）、Set（集合）、Hash（散列）、Zset（有序集合）。



（2）3种特殊数据结构：HyperLogLogs（基数统计）、BitMap（位存储）、Geospatial（地理位置）

#### 1.Redis 双写一致性：redis作为缓存，mysql的数据如何与redis进行同步呢？

这里有两种需求（1）一致性要求高的 （2）允许延迟一致，这里分开来描述：

* **一致性要求高的（但是性能不高）：**
  * 读数据的时候添加读锁（其他线程可以读，但是不能写，跟共享锁一个概念），写数据的时候添加写锁（其他线程既不能读，也不能写，类似于排他锁），读写锁可以从redission中获取。
* **允许延迟一致的（异步通知、基于Canal的异步通知）：**
  * ![image-20240406200855325](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240406200855325.png)
  * 基于Canal的异步通知![image-20240406201005207](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240406201005207.png)

无论是先更新数据库，再更新缓存；还是先更新缓存，再更新数据库，这两个方案都存在并发问题，当两个请求并发更新同一条数据的时候，可能会出现缓存和数据库中的数据不一致的现象。

**因此采用Cache Aside 策略，不更新缓存，而是删除缓存中的数据。最后到读取数据时，发现缓存中没有数据之后，再从数据库中读取数据，更新到缓存中。**

* **写策略的步骤：**
  * 更新数据库中的数据
  * 删除缓存中的数据
* **读策略的步骤：**
  * 如果读取的数据命中了缓存，则直接返回数据；
  * 如果读取的数据没有命中缓存，则从数据库中读取数据，然后将数据写入到缓存中，并返回给用户。

**针对写策略步骤的执行先后顺序，又会有一些问题：**

* 如果先删除缓存，再更新数据库的话，在**读+写**并发的时候，还是会出现缓存和数据库数据的不一致的情况；
  * 这种情况下为了解决一致性问题，我们采用**延迟双删**的方法，即**删除缓存，更新数据库，延迟一段时间后，再次删除缓存。**主要是为了确保请求A在睡眠的时候，请求B能够在这一段时间内完成从数据库读取数据，再把缺失的缓存写入缓存的操作，**但是具体睡眠多久比较玄学，搞不好依然会出现 读脏数据的问题。**
* 如果**先更新数据库，再删除缓存**，在并发的时候，还是会出现缓存和数据库的数据不一致的情况，**但是出现这种情况的概率很小，因为缓存的写入远快于数据库的写入，**很难出现请求B已经更新了数据库并删除了缓存后，请求A才更新完缓存的情况。**所以，这种情况是可以保证一致性的**
  * 但是这个方案因为在每次更新数据库的时候都要删除缓存，所以会导致缓存命中率降低。因此可以采用**更新数据库+ 更新缓存的方案**，但是会出现并发安全问题，因此有两种解决办法**（1）在更新缓存之前添加一个分布式锁，保证同一时间只有一个请求更新缓存；（2）更新完缓存之后，给缓存加上较短的过期时间，这样即使出现不一致的情况，数据也会很快过期。**





### 黑马点评

#### （1）商家缓存主动更新策略（根据id查询商家信息时用到）：

* 这一策略同时进行数据库操作和缓存操作，加入了事务，以保证同成功同失败。

* **先更新数据库，再更新缓存：** 缓存的读写是很快的，而数据库的操作是很慢的，如果先操作缓存（即先删除缓存），再更新数据库时，另一个线程在进行同样的查询的过程中，由于没有读到缓存，会去读数据库中的数据（而数据库数据更新很慢，这时还没更新完成），因此会读到旧数据，并把这个旧数据给写入缓存中，这就导致了缓存数据和数据库数据的不一致。![](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240320213306700.png)

* 具体实现:  

  ```java
  @Transactional
      //更新店铺信息
      public Result update(Shop shop) {
          Long id = shop.getId();
          if (id == null) {
              return Result.fail("店铺id 不能为空");
          }
          //1.更新数据库
          updateById(shop);
          //2.删除缓存
          stringRedisTemplate.delete(CACHE_SHOP_KEY + id);
          return Result.ok();
      }
  ```

  

#### （2）缓存穿透解决办法

* **缓存穿透是指，请求的数据在缓存和数据库中都不存在，这样缓存永远不会生效，这些请求都会打到数据库**

* **解决策略：**

  * 缓存空对象（本项目用的这个） ：数据库也查不到信息后，向缓存中缓存空数据（会设置TTL），防止一直访问数据库。
    * 优点：实现简单，维护方便
    * 缺点：因为缓存了空数据，造成了额外的内存消耗。同时，如果数据库又更新了数据，而此时缓存中仍为空数据，会造成短时间内的数据不一致（在缓存TTL结束后就可以保持一致了）。
  * 设置布隆过滤器：
    * 优点：内存占用比较少，没用使用多余的key。
    * 缺点：实现复杂，并且存在误判的可能。
  * 缓存预热时，需要预热布隆过滤器。
  

![image-20240320214835903](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240320214835903.png)

  * 关于布隆过滤器的原理：
    * 当我们向过滤器添加一个元素key的时候，我们通过多个hash函数，分别计算出多个值，然后将这些值所在的位置置为1.
    * **那么如果新来一个数据，如何判断它存不存在？**当新数据来的时候，同样使用前面的多个hash函数计算值，查看每一个计算结果对应的位置是不是1，如果有一个不是1，那么就认定这个数据不存在。
    * **如果每个位置都是1，也不能保证这个数据真的存在。**毕竟多个数据可以通过hash计算出相同的位置。如果这个位置是1，可能是另一个数据计算得来的。**因此，布隆过滤器可以判定某个数据一定不存在，但不能保证某个数据一定存在。**
    * 布隆过滤器的优缺点很明显：使用二进制数组，占用内存极少，并且插入和查询速度都很快；但是随着数据的增加，误判率会增加，无法判断数据一定存在；此外还有一个重要的缺点，就是他无法删除数据。

#### （3）缓存雪崩解决办法：

* **缓存雪崩**是指在同一时段内，大量的缓存key同时失效或者redis服务宕机，导致大量的请求打到数据库，带来巨大的压力。
* **解决策略： ** 
  * 给不同的key的ttl添加随机值
  * 利用redis集群提高服务的可用性
  * 为缓存业务添加降级限流策略：分库分表可还行。
  * 为业务设置多级缓存：例如使用本地缓存 + 分布式缓存的方式，不同级别缓存时间过时间不一样，即使某个级别缓存过期了，还有其他缓存级别兜底。

#### （4）缓存击穿解决办法：

* **缓存击穿**也叫热点key问题，就是一个**被高并发访问，并且缓存重建业务较复杂的key**突然失效了，无数的请求访问会在瞬间给数据库带来巨大冲击。

* 缓存雪崩是指大量的缓存key失效，而缓存击穿是指热点key失效。缓存穿透是指要查的key在缓存中和数据库中都不存在。

* （查询数据库和重建缓存的过程会很缓慢，在这期间如果有大量的请求过来，都是未命中，导致一直访问数据库。）![image-20240321202635871](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240321202635871.png)

* **解决策略：** 

  * 互斥锁：

    * 每个请求如果未命中缓存后需要进行缓存的重建工作，这时我们指定唯一获得互斥锁的线程进行缓存重建，等到重建完成后再释放锁。在此期间其他线程会重复缓存查询以及获取锁的过程。

    * 关于互斥锁的构建，这里使用stringRedisTemplate 的setIfAbsent来进行实现（Redis的Setnx命令），**在指定的key不存在时，为key设置指定的值。返回值表示设置成功与失败**

    * ```java
      // 获取互斥锁，返回值标识是否获得
          private boolean tryLock(String lockKey) {
              // 添加锁，并设置有效期，有效期时长要比业务时间长
              Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(lockKey, "1", 10L, TimeUnit.SECONDS);
              // 拆箱过程中可能会出现空指针异常，所以使用hutool提供的工具来解决
              return BooleanUtil.isTrue(flag);
          }
      
          // 释放锁
          private void unlock(String lockKey) {
              stringRedisTemplate.delete(lockKey);
          }
      ```

  * 逻辑过期

    * 逻辑过期并不是真正的过期，对于对应的key我们并不需要去设置TTL，而是通过业务逻辑来达到一个类似于过期的效果。其本质还是限制落到数据库的请求数量！
    * 这样一来，缓存基本是会被命中的，因为没有给缓存设置任何过期时间，并且对于key的设置都是实现选择好的，如果出现未命中的情况基本可以判断他不在选择之内，这样我们就可以直接返回错误信息。
    * 那么对于命中的情况，就需要先判断逻辑时间是否过期，再根据结果来决定是否进行缓存重建。**而这里的逻辑时间就是减少大量请求落到数据库的一个关口**

  * ![image-20240321202444133](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240321202444133.png)

  * 在发现逻辑过期后（意味着数据更新），首先尝试获取互斥锁，在获取到锁后，会开启新的线程来进行缓存的重建，而原线程返回的是未更新的老数据，这意味着使用**逻辑时间不保证数据的一致性。**

  * **注意做逻辑过期时，要先预热，手动缓存热点key**

  * 在进行商铺信息的查询的过程中，使用了互斥锁以及逻辑过期时间的方式解决缓存击穿问题。

  ```java
  /**
       * 使用互斥锁，解决缓存击穿问题
       * 缓存穿透问题：缓存穿透是指客户请求的数据在缓存中和数据库中都不存在，这样缓存永远不会生效，这些请求都会被打到数据库。
       * 缓存击穿问题：指热点key失效，大量请求打到数据库的问题。
       */
  
      private Shop queryWithMutex(Long id) {
          Shop shop;
          //1.从redis中查询商铺缓存
          String key = CACHE_SHOP_KEY + id;
          String shopJson = stringRedisTemplate.opsForValue().get(key);
          //2.判断是否存在
          if (StrUtil.isNotBlank(shopJson)) {
              //3.存在则直接返回
              shop = JSONUtil.toBean(shopJson, Shop.class);
              return shop;
          }
  
          // 因为我们在解决缓存穿透的时候会将空值放入redis中，所以可能会取得空值，因此需要对空值再进行一次判断
          if (shopJson != null) {
              // 命中的信息是空值，所以直接进行返回
              return null;
          }
          //添加互斥锁，解决缓存击穿问题
          //第一步，获取互斥锁
          String lockKey = RedisConstants.LOCK_SHOP_KEY + id;
          try {
              boolean isLock = tryLock(lockKey);
              //第二步，判断是否获得锁
              if (!isLock) {
                  //第三步，如果失败，进行休眠，休眠后重试
                  Thread.sleep(10);
                  return queryWithMutex(id);
              }
              /*第四步，如果成功，应再次检测redis缓存是否存在，做double check，这样做的原因是我们在获得锁之后，其他
               *线程可能已经将数据写入redis缓存中了，那么我们即使拿到锁，也没有必要再去查询数据库了，直接从redis中获取
               * 就可以了。
               * 而且，代码已经走到了这一步，说明我们的key取得的缓存不是我们用于解决缓存穿透时设置的空值了，因此不需要在进行
               * 空值的校验了。
               * */
              String secondShopJson = stringRedisTemplate.opsForValue().get(key);
              if (StrUtil.isNotBlank(secondShopJson)) {
                  // 缓存中存在，则直接返回
                  Shop secondShop = JSONUtil.toBean(shopJson, Shop.class);
                  return secondShop;
              }
              //走到这，说明拿到锁后，缓存中依旧没有出现我们想要的值，那么就需要查询数据库了
              shop = getById(id);
              // 模拟重建的延时
              Thread.sleep(200);
              //5.判断数据库中是否存在
              if (shop == null) {
                  // 避免缓存穿透问题，若数据库中不存在数据，则在redis中存入空值
                  stringRedisTemplate.opsForValue().set(key, "", RedisConstants.CACHE_NULL_TTL, TimeUnit.MINUTES);
                  //7.若不存在，则报错
                  return null;
              }
              //6.若存在，则存入redis缓存，并返回
              stringRedisTemplate.opsForValue().set(key, JSONUtil.toJsonPrettyStr(shop), CACHE_SHOP_TTL, TimeUnit.MINUTES);
          } catch (Exception e) {
              throw new RuntimeException(e);
          } finally {
              //释放锁
              unlock(lockKey);
          }
          return shop;
      }
  ```

  ```java
  //使用线程池来管理线程
      private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(20);
  
      private Shop queryWithLocalExpire(Long id) {
          Shop shop;
          //1.从redis中获取缓存
          String key = CACHE_SHOP_KEY + id;
          String shopJson = stringRedisTemplate.opsForValue().get(key);
          //2.判断是否为空，如果为空则返回null;
          if (StrUtil.isBlank(shopJson)) {
              return null;
          }
          //3.如果不为空，需要先把json反序列化为对象
          RedisData redisData = JSONUtil.toBean(shopJson, RedisData.class);
          JSONObject data = (JSONObject) redisData.getData();
          shop = JSONUtil.toBean(data, Shop.class);
          LocalDateTime expireTime = redisData.getExpireTime();
          //4.获取时间，判断是否过期
          if (LocalDateTime.now().isBefore(expireTime)) {
              //4.1 如果没有过期，则直接返回即可
              return shop;
          }
          //4.2 如果过期，需要进行缓存重建
          //5 缓存重建
          //6 获取互斥锁
          String lockKey = RedisConstants.LOCK_SHOP_KEY + id;
          //6.1 判断是否获取成功，如果成功，开启新线程进行缓存重建
          boolean isLock = tryLock(lockKey);
          if (isLock) {
              CACHE_REBUILD_EXECUTOR.submit(() -> {
                  try {
                      // 重建缓存
                      this.saveShop2Redis(id, 20L);
                  } catch (Exception e) {
                      throw new RuntimeException(e);
                  } finally {
                      // 释放锁
                      unlock(lockKey);
                  }
              });
          }
          //7 返回旧数据
          return shop;
      }
  ```

  <img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240321210833411.png" alt="image-20240321210833411" style="zoom:67%;" />

#### (5) 基于List的点赞列表&基于SortedSet的点赞排行榜

* 用户发表博文内容时，后端接收到的请求中不包含userID，userID需要我们从ThreadLocal中获取。在保存博文时，需要set博文的作者userID，在保存成功后，查询该用户的所有粉丝，并将博文信息推送给粉丝。

* 这里我们使用sortedSet来实现发布信息的推送，key为follow_user_id(粉丝ID)，value为blog_id，score为System.currentTimeMillis()。（这里可以优化为使用mq进行实现）

  ```java
  @Override
      public Result saveBlog(Blog blog) {
          //获取登录用户
          Long userId = UserHolder.getUser().getId();
          //保存blog
          blog.setUserId(userId);
          boolean save = save(blog);
          if (save) {
              //查询笔记作者的所有粉丝
              List<Follow> follows = followService.query().eq("follow_user_id", userId).list();
              for (Follow follow : follows) {
                  //获取粉丝id
                  Long followUserId = follow.getUserId();
                  String key = RedisConstants.FEED_KEY + followUserId;
                  //推送
                  stringRedisTemplate.opsForZSet().add(key, blog.getId().toString(), System.currentTimeMillis());
              }
              //推送笔记的id给所有粉丝
          }
          //返回blog id
          return Result.ok(blog.getId());
      }
  ```

* 对于博客实体类，我们通过'@TableField(exist = false)'注解来指定，icon（头像）,name（用户姓名）表示这两个字段不属于Blog表中的字段。

  ```java
  /**
       * 用户图标
       */
      @TableField(exist = false)
      private String icon;
      /**
       * 用户姓名
       */
      @TableField(exist = false)
      private String name;
      /**
       * 是否点赞过了
       */
      @TableField(exist = false)
      private Boolean isLike;
  ```

  

* 因为用户和头像这俩字段不属于blog的数据表字段，所以不仅在保存博客的时候我们需要从threadLocal中获取用户id并保存，在查询博文内容的时候，我们还要根据博文的user ID来查询发布者的用户名以及头像等信息。

* **点赞功能（首页探店排行榜&博文详情页都可以点赞）**

  点赞功能的需求如下：（1）同一个用户只能点赞一次，再次点击则取消点赞；（2）如果当前用户已经点赞，则点赞按钮高亮显示（前端已经实现了，判断字段Blog类的isLike属性）

  * 实现步骤：

    * 给Blog类中添加一个isLike字段，用来表示当前blog是否被当前用户点赞。

    * 修改点赞功能，利用Redis的set集合判断是否点赞过，未点赞过则点赞数加1，已点赞过则点赞数-1。缓存key为blog的id，缓存采用sortedSet的数据结构，value为userID，score为用户点赞时刻（**即，每个blogID对应的是所有点赞该blog的userid，userID根据用户点赞时间为score进行排序**）

      ```java
      @Override
          public Result likeBlog(Long id) {
              //实现帖子点赞功能
              // 1. 获取当前登录用户
              Long userId = UserHolder.getUser().getId();
              //2.判断当前用户是否已经点赞
              String key = RedisConstants.BLOG_LIKED_KEY + id;
              Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
              if (score == null) {
                  //3.如果未点赞，可以点赞
                  //3.1 数据库点赞数加一
                  boolean isSuccess = update().setSql("liked = liked + 1").eq("id", id).update();
                  //3.2 保存点赞用户id 到redis sortedSet集合
                  if (isSuccess) {
                      stringRedisTemplate.opsForZSet().add(key, userId.toString(), System.currentTimeMillis());
                  }
              } else {
                  //4. 如果已经点赞，取消点赞
                  //4.1 数据库点赞数减一
                  boolean isSuccess = update().setSql("liked = liked - 1").eq("id", id).update();
                  //4.2 将用户信息从reids sortedSet集合中删除
                  if (isSuccess) {
                      stringRedisTemplate.opsForZSet().remove(key, userId.toString());
                  }
              }
              return Result.ok();
          }
      ```

      

    * 修改queryBlogById方法，判断当前登录用户是否点赞过（即判断能不能查到当前userID，有就说明已点赞），并将结果赋值给blog的isLike字段。(**isLike字段不属于blog表字段，它的作用是，当前用户如果点赞过，那么就将isLike置为真，在客户端发请求查询blog信息时，前端根据这个字段决定是否高亮显示点赞按钮**)

      ```java
       public void isBlogLiked(Blog blog) {
              //判断当前笔记是否被点赞过
              UserDTO user = UserHolder.getUser();
              if (user == null) {
                  //用户未登录，不用关心是否点赞过，直接返回
                  return;
              }
              Long userId = user.getId();
              String key = RedisConstants.BLOG_LIKED_KEY + blog.getId();
              Double score = stringRedisTemplate.opsForZSet().score(key, userId.toString());
              blog.setIsLike(score != null);
          }
      ```

      

    * 修改分页查询Blog业务，判断当前用户是否点赞过，赋值给isLike字段（**这个业务是用于分页展示点赞排行榜的博文，点赞排行榜根据博文的点赞数量降序排序并分页展示的**）。

      ```java
      public Result queryHotBlog(Integer current) {
              // 根据用户查询
              Page<Blog> page = blogService.query()
                      .orderByDesc("liked")
                      .page(new Page<>(current, SystemConstants.MAX_PAGE_SIZE));
              // 获取当前页数据
              List<Blog> records = page.getRecords();
              // 查询用户,以及查询是否被点赞过
              records.forEach(blog -> {
                  this.queryBlogUser(blog);
                  this.isBlogLiked(blog);
              });
              return Result.ok(records);
          }
      ```

    * 查询当前帖子被哪些用户点赞了（**这里只在前端展示前5位点赞用户的头像，按照点赞时间从前到后进行排序**）

      ```java
      public Result queryBlogLikes(Long id) {
              //返回帖子点赞top5用户
              String key = RedisConstants.BLOG_LIKED_KEY + id;
              Set<String> top5 = stringRedisTemplate.opsForZSet().range(key, 0, 4);
              if (top5 == null || top5.isEmpty()) {
                  return Result.ok();
              }
              //2. 解析出其中的用户id
              List<Long> ids = top5.stream().map(Long::valueOf).collect(Collectors.toList());
              String idListStr = StrUtil.join(",", ids);
              //3.根据用户id查询用户，返回已点赞用户列表
              //List<UserDTO> users = userService.listByIds(ids).stream().map(user -> BeanUtil.copyProperties(user, UserDTO.class)).collect(Collectors.toList());
              List<UserDTO> users = userService.query().in("id", ids).last("order by field(id," + idListStr + ")").list()
                      .stream()
                      .map(user -> BeanUtil.copyProperties(user, UserDTO.class))
                      .collect(Collectors.toList());
              //4.返回
              return Result.ok(users);
          }
      ```


#### （6）好友关注功能

* 在探店图文的详情页面中，可以关注发布笔记的作者：

* **业务逻辑：**

  * **关注用户**功能的实现：前端传一个isFollow值，如果用户已经关注该博主，则isFollow为false，没有关注资格，但可以进行取关。反之为true，说明可以进行关注。new Follow对象，将user ID和userFollowID（用户关注的人）都set到Follow对象里。再将follow对象save到数据库表中。

  * ```java
    public Result follow(Long id, boolean isFollow) {
            //关注和取关
            //1. 获取登录用户
            UserDTO user = UserHolder.getUser();
            if (user == null) {
                //如果用户未登录，不用判断是否关注
                return Result.ok();
            }
    
            Long userId = user.getId();
            String key = RedisConstants.FOLLOW_USER_KEY + userId;
            if (isFollow) {
                // 如果未关注，新增数据
                Follow follow = new Follow();
                follow.setUserId(userId);
                follow.setFollowUserId(id);
                boolean isSuccess = save(follow);
                if (isSuccess) {
                    stringRedisTemplate.opsForSet().add(key,id.toString());
                }
            } else {
                //如果已关注，取关删除数据
                boolean isSuccess = remove(new QueryWrapper<Follow>().eq("user_id", userId).eq("follow_user_id", id));
                if (isSuccess) {
                    stringRedisTemplate.opsForSet().remove(key,id.toString());
                }
            }
            return Result.ok();
        }
    ```

  * **查看他人主页功能的实现：**前端传该人的userID ，首先根据user ID查询用户信息，其次还要查询该用户发表的博文信息，并分页进行展示：

  * ```java
    // 根据id查询用户
        @GetMapping("/{id}")
        public Result queryUserById(@PathVariable("id") Long userId){
            // 查询详情
            User user = userService.getById(userId);
            if (user == null) {
                return Result.ok();
            }
            UserDTO userDTO = BeanUtil.copyProperties(user, UserDTO.class);
            // 返回
            return Result.ok(userDTO);
        }
    @GetMapping("/of/user")
    public Result queryBlogByUserId(
    		@RequestParam(value = "current", defaultValue = "1") Integer current,
    		@RequestParam("id") Long id) {
    	// 根据用户查询
    	Page<Blog> page = blogService.query()
    			.eq("user_id", id).page(new Page<>(current, SystemConstants.MAX_PAGE_SIZE));
    	// 获取当前页数据
    	List<Blog> records = page.getRecords();
    	return Result.ok(records);
    }
    ```

  * **共同关注功能的实现：**共同关注功能使用redis中的**set结构**进行实现，求两个用户关注列表的交集，因此我们不仅需要吧数据存入follow表中（保存关注关系），还要把user ID放入redis的set集合中。

    ```java
    public Result followCommons(Long id) {
            //获取当前用户
            UserDTO user = UserHolder.getUser();
            if (user == null) {
                //如果用户未登录，不用判断是否关注
                return Result.ok();
            }
            Long userId = user.getId();
    
            String key1 = RedisConstants.FOLLOW_USER_KEY + userId;
            String key2 = RedisConstants.FOLLOW_USER_KEY + id;
            //求交集
            Set<String> intersect = stringRedisTemplate.opsForSet().intersect(key1, key2);
            //查询交集中的用户 信息
            if (intersect == null || intersect.isEmpty()) {
                return Result.ok(Collections.emptyList());
            }
            List<Long> ids = intersect.stream().map(Long::valueOf).collect(Collectors.toList());
            List<UserDTO> commons = userService.listByIds(ids)
                    .stream().map(followUser -> BeanUtil.copyProperties(followUser, UserDTO.class))
                    .collect(Collectors.toList());
            return Result.ok(commons);
        }
    ```

* **Feed流实现关注用户的博文推送：**

  * 本例中的个人界面，是基于关注的好友来进行feed流的，因此采用timeLine的方式，依照时间顺序刷新。（也有像抖音那种的，使用智能推荐算法进行推荐的智能排序的feed流）。

  * 实现方案有三种：**推模式、拉模式、推拉结合的模式（推拉是相对于当前用户而言的）**

    * 拉模式：又称为读扩散，每个博主发送信息的时候都发送到自己的发件箱中（一个缓冲区），用户读的时候从各个关注用户的发件箱中读取信息。但是读操作过于频繁，若用户关注了许多博主，一次要读的消息也是十分的多，延迟会很高。

    ![image-20240326203018064](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240326203018064.png)

    * 推模式：又称为写扩散，每个用户有自己的收件箱（一个缓冲区），每个博主发送信息的时候，将发送内容推送至每一个用户的收件箱中。显然，这样会导致写操作十分频繁，如果博主有许多粉丝，写操作会更频繁。

    * ![image-20240326203034238](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240326203034238.png)

    * 推拉结合的模式：也称作读写结合，兼具推和拉两种方式的优点。对于普通博主，粉丝少，那么就可以使用推模式，这样写操作就并不繁重；对于大博主，粉丝多，分为两种情况，活跃粉多，那么就可以使用拉模式；活跃粉少，那么就可以使用推模式。

      ![image-20240326203318389](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240326203318389.png)

* **推送到粉丝收件箱：**

  * 需求：

    * 修改新增探店笔记的业务，在保存blog到数据库的同时，推送到粉丝的收件箱（收件箱使用sortedSet实现）
    * 收件箱满足可以根据时间戳排序，必须用redis的数据结构实现。
    * 查询收件箱数据的 时候，可以进行分页查询。

  * **注意：Feed流中的数据会不断更新，所以数据的角标也在变化，因此不能采用传统的分页模式。**

    ![image-20240326203940897](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240326203940897.png)

    ![image-20240326204001342](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240326204001342.png)



* 新增博文信息的实现：

  ```java
   public Result saveBlog(Blog blog) {
          //获取登录用户
          Long userId = UserHolder.getUser().getId();
          //保存blog
          blog.setUserId(userId);
          boolean save = save(blog);
          if (save) {
              //查询笔记作者的所有粉丝
              List<Follow> follows = followService.query().eq("follow_user_id", userId).list();
              for (Follow follow : follows) {
                  //获取粉丝id
                  Long followUserId = follow.getUserId();
                  String key = RedisConstants.FEED_KEY + followUserId;
                  //推送
                  stringRedisTemplate.opsForZSet().add(key, blog.getId().toString(), System.currentTimeMillis());
              }
              //推送笔记的id给所有粉丝
          }
          //返回blog id
          return Result.ok(blog.getId());
      }
  ```

* 滚动分页的实现：**简单来说，每次查询，我们都记住上一次查询的score的最小值，然后从小于最小值的第一个值进行查询**，核心代码如下：

  ```java
  String key = RedisConstants.FEED_KEY + userId;
          Set<ZSetOperations.TypedTuple<String>> typedTuples = stringRedisTemplate.opsForZSet().reverseRangeByScoreWithScores(key, 0, max, offset, 2L);
  ```

  对应的redis指令是`ZrevrangeByScore 【key]】【上次查到的score最小值】【score的最小值】 withscores Limit 【偏移量，这里为1】，【count，页面大小】`。

  翻译一下就是：以上次查询的最小值作为本次查询range的max，min为0，limit指定从上次查询的最小值的下一个元素（为0的话是查询<=最大值的的元素，会造成重复，所以这里应该为1，查询<=最大值的最近的元素）进行查询（根据偏移量来的），总共查询count个元素。

  这里计算最小时间minTime和偏移量offset的代码也很巧妙：

  ```java
  long minTime = 0;
          int ofCount = 1;
          List<String> ids = new ArrayList<>(typedTuples.size());
          //3. 解析收件箱：blogId ,score ,minTime(时间戳)
          for (ZSetOperations.TypedTuple<String> typedTuple : typedTuples) {
              ids.add(typedTuple.getValue());
              long time = typedTuple.getScore().longValue();
              if (time == minTime) {
                  ofCount ++;
              } else {
                  ofCount = 1;
                  minTime = time;
              }
          }
  ```

  **概括一下就是，ofCount是用来计算score值等于最小时间的元素个数，后续作为offset封装到对象中，time在遍历过程中不断进行更新，遍历完就是取得了最小值，把它付给minTime，封装到对象中去。**

  在查询后，又会将本次查询的最小值和偏移量封装到ScrollResult对象中，供下一次查询使用。

  总体代码如下：

  ```java 
  @Override
      public Result queryBlogOfFollow(Long max, Integer offset) {
          //1. 获取当前登录用户
          Long userId = UserHolder.getUser().getId();
          //2. 获取收件箱
          String key = RedisConstants.FEED_KEY + userId;
          Set<ZSetOperations.TypedTuple<String>> typedTuples = stringRedisTemplate.opsForZSet().reverseRangeByScoreWithScores(key, 0, max, offset, 2L);
          // 非空判断
          if (typedTuples.isEmpty() || typedTuples == null) {
              return Result.ok();
          }
          long minTime = 0;
          int ofCount = 1;
          List<String> ids = new ArrayList<>(typedTuples.size());
          //3. 解析收件箱：blogId ,score ,minTime(时间戳)
          for (ZSetOperations.TypedTuple<String> typedTuple : typedTuples) {
              ids.add(typedTuple.getValue());
              long time = typedTuple.getScore().longValue();
              if (time == minTime) {
                  ofCount ++;
              } else {
                  ofCount = 1;
                  minTime = time;
              }
          }
          String idStr = StrUtil.join(",", ids);
          //4. 根据id查询blog
          List<Blog> blogs = blogService.query().in("id", ids).last("order By field (id," + idStr + ")").list();
          //4.1 查询blog用户，blog是否被点赞过
          for (Blog blog : blogs) {
              queryBlogUser(blog);
              isBlogLiked(blog);
          }
          //封装并返回
          ScrollResult scrollResult = new ScrollResult();
          scrollResult.setList(blogs);
          scrollResult.setMinTime(minTime);
          scrollResult.setOffset(ofCount);
          return Result.ok(scrollResult);
      }
  ```

  

  

#### （7）优惠券秒杀

* **全局ID生成器**：是一种在分布式系统下用来生成全局唯一id的工具，满足以下特性

  * **唯一性**
  * 高可用
  * 高性能
  * **递增性**
  * **安全性**

* 我们生成的ID由以下部分组成，订单ID就使用了其生成的ID。

  ![image-20240329215303515](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240329215303515.png)

* 具体的工具类代码如下：

  ```java
  // prefix是业务的前缀，我们可以为不同的业务使用不同的全局ID
  public long nextId(String keyPrefix){
          //1. 生成时间戳（31位）
          long timeStamp = LocalDateTime.now().toEpochSecond(ZoneOffset.UTC) - BEGIN_TIMESTAMP;
          //2. 生成序列号
          //2.1 获取当前日期，精确到天
          String date = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
          //2.2 自增长（添加了date是为了防止key自增到上限，并且方便统计）
          Long count = stringRedisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);
          //3. 拼接并返回
          return (timeStamp << COUNT_BITS) | count;
      }
  ```

  **注意拼接这一步，`timeStamp <<  32`是先让时间戳左移32位，然后与序列号进行或运算**

* **乐观锁解决超卖问题**

  * 超卖问题产生的原因：线程1查询库存，发现余额为1并且大于0，可以进行扣减，但是在线程1进行扣减库存操作之前，有一个线程2，它也查询库存，发现为1(因为线程1还没开始扣减)，于是它也进行扣减，这就导致了1个库存却被两个线程消费了，这就是超卖问题。

  * **乐观锁&悲观锁**：

    * 悲观锁认为线程安全问题一定会发生，因此在**操作数据之前都要先获取锁**，确保线程串行执行。（例如Synchronize、Lock都属于悲观锁）；
    * 乐观锁则认为，线程安全问题不一定会发生，因此不进行加锁，只是**在数据进行更新的时候去判断有没有其他线程对数据做了修改**。如果没有修改，则认为是安全的，自己可以更新数据；如果已经被其他线程修改，说明发生了安全问题，此时可以重试或者异常。

  * 悲观锁实现比较简单，操作前获取锁，操作结束后才释放锁，让多个线程串行执行，但是你让并发线程串行执行，效率会十分低下。

  * **乐观锁设计：**

    * 第一种设计：**添加版本号version**，在扣减操作前会读取库存stack和版本号version，在进行扣减时，要检查version的值是不是最开始查到的值，如果是的话，才进行stack的扣减，并且让version自增；如果version与最开始查到的值不同，说明有其他线程修改过库存了，此时应该进行重试操作。

      <img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240330150853650.png" alt="image-20240330150853650" style="zoom: 50%;" />

    * 第二种设计：**CAS法**，这里我们直接对比库存值，在进行扣减操作前会查询库存stack值，在进行扣减操作时，检查statck值（**内存中的值V**）是否与最开始读到的值（**预期原始值A**）相同，如果相同则可以进行扣减，反之，不同的话说明有其他线程更改过库存了，此时应该进行重试工作。

      ```
      什么是CAS机制（compare and swap）
      CAS算法的作用：解决多线程条件下使用锁造成性能损耗问题的算法，保证了原子性，这个原子操作是由CPU来完成的
      CAS的原理：CAS算法有三个操作数，通过内存中的值（V）、预期原始值（A)、修改后的新值。
      （1）如果内存中的值和预期原始值相等， 就将修改后的新值保存到内存中。
      （2）如果内存中的值和预期原始值不相等，说明共享数据已经被修改，放弃已经所做的操作，然后重新执行刚才的操作，直到重试成功。
      ```

    * 创建订单功能采用CAS法的实现：

      ```java
      //5.扣减库存
              boolean success = seckillVoucherService.update()
                      .setSql("stock = stock - 1")//set stock = stock -1
                      .eq("voucher_id",voucherId).eq("stock",voucher.getStock()) //where id = ? and stock =?
                      .update();
      ```

      上面代码的意思就是，在执行update的时候，再次检查stock值和当前获取到的stack值是否相同。**但是这样修改后，我们发现成功率很低，在开启200个线程执行的情况下，100张只卖出去了23张。**原因就在于，在并发环境下，stock值前后匹配的成功率很小，这导致大量的线程购买失败了。

      因此对上面的代码继续修改，在扣减前查询stack值是否大于0，如果大于0，当前线程就可以进行扣减库存的操作了，如果等于0，说明有其他线程已经把库存扣完了，于是返回false表示库存扣减失败。

      ```java
      // 5.扣减库存
              boolean success = seckillVoucherService.update()
                      .setSql("stock = stock - 1")
                      .eq("voucher_id", voucherId).gt("stock", 0)
                      .update();
      ```

* **一人一单问题**

  * 需要修改业务逻辑，一名用户只能购买一张优惠券，防止黄牛刷券。

  * 具体业务流程如下：

    ![image-20240330154534430](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240330154534430.png)
    
  * 于是修改代码如下：
  
    * ```java
      //查询订单
              int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
              if (count > 0) {
                  return Result.fail("不可重复下单！");
              }
              // 5.扣减库存
              boolean success = seckillVoucherService.update()
                      .setSql("stock = stock - 1")
                      .eq("voucher_id", voucherId).gt("stock", 0)
                      .update();
              if (!success) {
                  return Result.fail("优惠券库存不足！");
              }
              // 6.创建订单
      	。。。。。。。
      ```
  
      **问题1**但是我们在测试的时候，发现问题并没有解决，原因就在于，在高并发环境下，多个线程查询该userID下的订单数count，都会得到0，从而都进行库存的扣减。导致没有实现一人一单。
  
      那么自然考虑要加锁了，但是这里并不能用乐观锁方案，因为我们这里是新增数据操作，不是更新操作，数据一开始是不存在的。所以这里要使用悲观锁。
  
      那么这里将查询订单到创建订单这个过程为一个方法，给这个方法加锁。
  
      ```java
          @Transactional
          public synchronized Result createVoucherOrder(Long voucherId) {
              // 一人一单
              Long userId = UserHolder.getUser().getId();
              //synchronized (userId.toString().intern()) {
              //查询订单
              int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
              if (count > 0) {
                  return Result.fail("不可重复下单！");
              }
              // 5.扣减库存
              boolean success = seckillVoucherService.update()
                      .setSql("stock = stock - 1")
                      .eq("voucher_id", voucherId).gt("stock", 0)
                      .update();
              if (!success) {
                  return Result.fail("优惠券库存不足！");
              }
              // 6.创建订单
              long orderId = redisIdWorker.nextId("order");
              VoucherOrder voucherOrder = new VoucherOrder();
              //6.1订单id
              voucherOrder.setId(orderId);
              //6.2用户id
              voucherOrder.setUserId(userId);
              //6.3优惠券id
              voucherOrder.setVoucherId(voucherId);
              //7.保存订单
              save(voucherOrder);
              // 8.返回订单id
              return Result.ok(orderId);
          }
      ```
  
      **问题二，缩小锁范围：**但是在方法上使用synchronized锁的是当前this对象，导致每个用户到这里都要获取这一把锁，让这个方法串行执行了，效率会极低。我们只打算锁当前用户，不让他多次创建订单就可以了，那么这里我们synchronized（user ID.toString()）就可以了。
  
      **问题三，但是用了toString还是不好使，因为toString底层都是new了一个新的对象，所以锁无法起作用。**于是我们再次修改为`synchronized(userID.toString().intern())`，intern会从字符串常量池中寻找值一样的引用并返回。
  
      **问题四，但是，我们这里将锁定义在了方法内部，因此在锁释放之后，创建订单的事务才会提交。而在锁释放到事务提交这段时间内（这个时候，由于事务尚未提交，数据库还没更新，如果此时有另外的线程进入的话，依然查不到count，会继续进行订单创建，这是我们不能允许的），因此synchronize不能放在方法内。**于是就将代码改造成了下面这样：
  
      ```java
      public Result seckillVoucher(Long voucherId) {
              //1. 查询优惠券
              SeckillVoucher seckillVoucher = seckillVoucherService.getById(voucherId);
      
              // 2.判断秒杀是否开始
              LocalDateTime beginTime = seckillVoucher.getBeginTime();
              LocalDateTime endTime = seckillVoucher.getEndTime();
      
              if (LocalDateTime.now().isBefore(beginTime)) {
                  return Result.fail("秒杀活动尚未开始！");
              }
              // 3.判断秒杀是否已经结束
              if (LocalDateTime.now().isAfter(endTime)) {
                  return Result.fail("秒杀活动已经结束！");
              }
              // 4.判断库存是否充足
              Integer stock = seckillVoucher.getStock();
              if (stock <= 0) {
                  return Result.fail("优惠券已售空！");
              }
          
          // 注意这里，这里是我们修改的地方
              Long userId = UserHolder.getUser().getId();
           *  synchronized (userId.toString().intern()) {        
           *     return createVoucherOrder(voucherId);
           *  }
          }
      ```
  
      **问题五，但是这里还存在一个事务的问题**，这里return的实际上是this.createVoucherOrder()方法，而spring事务生效是因为spring对当前对象做了动态代理，用的是代理对象做事务处理，而这里用this，是一个非代理对象，显然事务是不会生效的。
  
      解决方案就是，拿到当前对象的代理对象。确保事务生效。
  
      **问题六，在分布式情况下，上面的一人一单解决方法不好使了。这里引出》》》》分布式锁！！**
  
      问题的原因如下，分布式情况下，有多个jvm存在，不同的jvm有多个锁监视器，这会导致每个jvm都会成功一个。
  
      ![image-20240330215139748](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240330215139748.png)
  
* **分布式锁：**

  * 分布式锁就是满足分布式系统或集群模式下多进程可见并且互斥的锁。
  * 分布式锁应有如下功能特性：**多进程可见、互斥、高可用、高性能、安全性**
  * 分布式锁的实现：
  * ![image-20240330220500961](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240330220500961.png)

* **Redis分布式锁误删问题：**

  ![image-20240331145142125](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240331145142125.png)

  **问题：**线程1在获取到锁，在进行自己的业务逻辑的时候却发生了阻塞，阻塞时间甚至超过了我们预设的redis锁超时时间。这导致锁提前被释放掉了，这时线程2自然而然获得了锁，在它执行业务逻辑的时候，**线程1的业务完成了，于是线程1二话不说，进行释放锁，**于是线程2的锁也没了，线程3就能获取锁执行自己的业务了.........

  **解决方法：**在释放锁的时候判断一下线程标识是否一致，如果一致，就释放锁，不一致就不释放。（很好判断，因为我们setNx的value就是线程id，但是在集群模式下，多个jvm会出现线程id的重复，因此我们存放的标识也要尽可能的唯一，这里采用UUID + 线程ID的方式。）

* **分布式锁的原子性问题**：

  ![image-20240331152244051](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240331152244051.png)

  上面我们解决了分布式锁误删问题，但是在并发情况下还是会存在隐患。

  **问题：**线程1在获取锁执行完业务进行释放的时候，由于我们释放锁的过程分为两步（1）判断线程标识是否一致（2）delete KEY，**而在完成第一步后释放锁之前，假设线程1阻塞了（很有可能发生，比如垃圾回收会阻塞所有线程），等到锁被超时释放掉的时候，**线程2可以趁虚而入了，它可以获取锁并执行自己的业务，**而在它执行业务的时候，线程1恢复了，由于线程1已经进行了一致性判断，因此线程1会直接把锁释放掉，**加入此时又来一个线程3，它就又可以执行自己的业务了。

  **解决方法：**这里我们使用lua脚本来保证释放锁的原子性。具体如下所示：

  ```lua
  -- 获取锁中的线程标识
  local id = redis.call('get',KEYS[1])
  -- 判断是否一致
  if (ARGV[1] == id)
  then
      return redis.call('del',KEYS[1])
  end
  return 0
  ```

  这里，为了避免每次释放锁的时候都要用IO流读脚本，这里采用静态代码块的方式，直接预加载我们的脚本。

  ```java
  //为了避免每次释放锁都要重新加载脚本浪费性能，所以我们选择使用静态代码块进行初始化加载
      private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
      static {
          UNLOCK_SCRIPT = new DefaultRedisScript<>();
          UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
          UNLOCK_SCRIPT.setResultType(Long.class);
      }
  ```

  这样一来，在类初始化后，我们的脚本也完成了预加载。

* **分布式锁工具类SimpleRedisLock**

  在解决了锁误删问题以及原子性问题后，我们的工具类如下所示：

  ```java
  @Override
      public boolean tryLock(long timeoutSec) {
          //获取当前线程标识
          String threadId = ID_PREFIX +Thread.currentThread().getId();
          //获取锁
          Boolean isSuccess = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
          return BooleanUtil.isTrue(isSuccess);
      }
  
      @Override
      public void unlock() {
         // 调用lua脚本, 参数列表： lua脚本，key,args
          stringRedisTemplate.execute(UNLOCK_SCRIPT, Collections.singletonList(KEY_PREFIX + this.name),ID_PREFIX +Thread.currentThread().getId());
      }
  ```

  

* **Redis分布式锁优化**

  * 基于setnx实现的分布式锁存在下面的问题**（引出Redission）**：
    * **不可重入：**同一个线程无法多次获取同一把锁。
    * **不可重试**： 获取锁只尝试1次就返回了，没有重试机制。
    * **超时释放**：锁超时释放虽然可以避免死锁，但如果是业务执行耗时较长，也会导致所释放，存在安全隐患。
    * **主从一致性：**如果redis提供了主从集群，主从同步存在延迟，当主宕机时，如果从同步主中锁数据，则会出现锁实现。
  * **Redission：**redission是一个在redis基础上实现的java驻内存数据网格。它**不仅提供了一系列的分布式地常用Java对象，还提供了许多分布式服务，其中就包含了各种分布式锁的实现。**（redis下各种分布式工具集合）

* **Redission入门：**

  * ![image-20240401141124091](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240401141124091.png)![image-20240401142605710](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240401142605710.png)![image-20240401143311866](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240401143311866.png)![image-20240401150536638](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240401150536638.png)
  * **Redission分布式锁原理：**
    * 可重入：利用Hash结构，记录线程id和重入次数
    * 可重试：利用信号量和PubSub功能实现等待、唤醒、获取锁失败的重试机制
    * 超时续约：利用watchDog，每隔一段时间（releaseTime / 3），重置超时时间
  * **Redission分布式锁主从一致性问题（联锁MultiLock）：**
    * 一般redis集群会设置主从模式，主节点用来写，从节点用来读，同时设置主从同步机制保证数据的一致性。但是如果我们从主节点获取锁，但是在主从同步过程中主节点宕机，这就会导致新的主节点没有锁的内容，产生并发安全问题。
    * Redission的解决方法是，不区分主从节点，所有节点同一级别，互不联系。在获取锁的时候，从每个redis节点获取锁，并且，只有当所有节点都能获取锁的时候该线程才能获取锁，这样就解决了分布式锁主从一致性问题。![image-20240401153807053](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240401153807053.png)

* **总结：**

  * （1）不可重入Redis分布式锁:
    * 原理：利用setNx的互斥性；利用ex避免死锁；释放锁时判断线程标识。
    * 缺陷：不可重入、无法重试、锁超时失效。
  * （2）可重入的Redis分布式锁：
    * 原理：利用hash结构，记录线程标识可重入次数；利用watchDog延续锁时间；利用信号量控制锁充实等待
    * 缺陷：redis宕机引起锁失效问题
  * （3）Redission的multilock:
    * 原理：多个独立的redis节点，必须在所有节点都获取重入锁，才算获取锁成功
    * 缺陷：运维成本高、实现复杂
  
* **异步秒杀思路**

  * 在我们使用了分布式锁改造后的优惠券秒杀过程可以概括为如下几步：

    * 根据传入的voucher ID查询优惠券信息（**访问数据库**）；
    * 判断秒杀是否开始或者已经结束（访问优惠券属性）
    * 判断库存是否充足（访问优惠券属性）
    * 获取当前用户id，查询当前用户是否已经购买过该优惠券（**访问数据库**）
    * 如果符合一人一单，那么进行库存扣减（**访问数据库**）
    * 创建订单信息并持久化（**访问数据库**）

  * **该过程涉及4次数据库访问，串行执行的时候，会相当消耗时间，从而造成效率的降低，因此需要对整个流程进行异步化改造。**

  * 这里，将判断秒杀库存和校验一人一单的功能交给redis来做，开启新的线程来完成那些需要访问数据库的操作。redis和tomcat之间通过一个消息队列来进行消息传递，当库存充足并且通过一人一单的校验后，redis向消息队列中发布消息，包含用户id，优惠券id，订单id；tomcat在接收到消息队列中的信息后，访问数据库，完成订单的创建。

  * ![image-20240402140309308](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240402140309308.png)

  * **缓存设计：**对于判断秒杀库存的缓存设计，直接使用string结构就可以；而对于一人一单的校验，这里我们使用set结构来保存。1是因为同一张id的优惠券可以被多个用户购买，2是这些value不能有重复的，因此选用set结构正合适。

    ![image-20240402141239535](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240402141239535.png)

  * 使用lua脚本来保证我们使用缓存进行库存判断和一人一单校验时的原子性：

    ![image-20240402141618424](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240402141618424.png)

  * 整个工作流程如下：

    <img src="https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240402141928196.png" alt="image-20240402141928196" style="zoom:67%;" />

  * 具体代码如下：

    ```java
     public Result seckillVoucher(Long voucherId) {
            Long userId = UserHolder.getUser().getId();
            
            //1. 执行lua脚本
            Long result = stringRedisTemplate.execute(
                    SECKILL_SCRIPT,
                    Collections.emptyList(),
                    voucherId.toString(), userId.toString(), String.valueOf(orderId));
            //2.判断结果，
            int r = result.intValue();
            if (r != 0) {
                //2.1 不为0，代表没有
                return Result.fail(r == 1 ? "库存不足！" : "不能重复下单！");
            }
         // 生成订单id
            long orderId = redisIdWorker.nextId("order");
         // 添加阻塞队列
         .......
         //3.返回订单id
            return Result.ok(orderId);
        }
    ```

  * 使用阻塞队列的方式完成异步下单 有一个很重要的缺点，那就是**阻塞队列使用的是jvm内存，在高并发情况下会有无数的对象放在阻塞队列中，这可能会导致内存的溢出；同时，由于内容都是存在于内存中，如果发生宕机，内存中的数据发生丢失，会出现数据的安全问题**（引出消息队列）。

  * 基于PubSub的消息队列的优缺点：

    * 优点：采用发布订阅模型，支持多生产，多消费
    * 缺点：不支持持久化；无法避免消息丢失；消息堆积有上限，超出时数据丢失

  * **基于Stream的消息队列--单消费模式：**

    * stream是redis5.0引入的一种新数据类型，可以实现一个功能非常完善的消息队列
    * stream中的消息读完不会删除。
    * ![image-20240403140103440](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240403140103440.png)
    * ![image-20240403140208717](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240403140208717.png)
    * ![image-20240403140306245](https://jiawei-blog-pictures.oss-cn-beijing.aliyuncs.com/blog_pictures/image-20240403140306245.png)
    * **阻塞方式的一个bug：**当我们指定起始id为$时，代表读取最新的消息，如果我们处理一条消息的过程中，又有超过1条以上的消息到达队列，则下次获取时也只能读取到最新的一条，会出现漏读消息的问题。
    * **总结，Stream类型的消息队列的xRead命令特点如下：**
      * 消息可回溯
      * 一个消息可以被多个消费者读取
      * 可以阻塞读取
      * 有消息漏读的风险

  * **基于Stream的消费队列--消费者组**
    * **消费者组：**将多个消费者划分到一个组中，监听同一个队列。具备以下特点：
      * **消息分流：**队列中的消息会分流给组内的不同消费者，而不是重复的消费，从而加快消息处理的速度
      * **消息标识：**消费者组会维护一个标识，记录最后一个被处理的消息，哪怕消费者宕机重启，还会从标识之后读取消息。确保每一个消息都会被消费
      * **消息确认：**消费者获取消息后，消息处于pending状态，并存入一个pending-List。当处理完成后需要XACK来确认消息，标记消息已处理，才会从pendingList中移除。
    * **Stream类型消息队列的XREADGROUP命令特点：**
      * 消息可回溯
      * 可以多消费者争抢消息，加快消费速度
      * 可以阻塞读取
      * 没有消息漏读的风险
      * 有消息确认机制，保证消息至少被消费一次


  * **基于Stream消息队列实现**

    * 流程变更如下：

      * 创建一个stream类型的消息队列，命名为stream.orders
      * 修改之前的秒杀下单lua脚本，在认定有抢购资格后，直接向stream.order中添加消息，内容包含voucherID、user ID、orderID
      * 项目启动时，开启一个线程任务，尝试获取stream.order中的信息，完成下单

    * 修改后的Lua脚本：

      ```lua
      --参数列表
      --优惠券id
      local voucherId = ARGV[1]
      --用户id
      local userId = ARGV[2]
      --订单id
      local orderId = ARGV[3]
      
      --数据key
      --库存key
      local stockKey = 'seckill:stock:' .. voucherId
      --订单key(这里的orderKey代表的是：该优惠券下的所有订单，它是一个集合的key)
      local orderKey = 'seckill:order:' .. voucherId
      
      --3.脚本业务
      --3.1判断库存是否充足 get stockkey
      if (tonumber(redis.call('get',stockKey)) <= 0) then
          --3.2 库存不足，返回1
          return 1;
      end
      --判断用户是否下单 sismember orderKey userId
      if (redis.call('sismember', orderKey, userId) == 1) then
          --3.3 存在，说明是重复下单，返回2
          return 2
      end
      --3.4 扣减库存，
      redis.call('incrby',stockKey,-1)
      --3.5 下单，保存用户(将用户id加入到以orderKey 为key的集合中)
      redis.call('sadd',orderKey,userId)
      --3.6 发送消息到消息队列中，XADD stream.orders * k1 v1 k2 v2....
      redis.call("XADD","stream.orders","*","userId",userId,"voucherId",voucherId,"id",orderId)
      return 0
      ```

    * 修改后的，使用stream的订单处理线程：

      ```java
      @PostConstruct
          private void init() {
              SECKILL_ORDER_EXECUTOR.submit(new VoucherOrderHandler());
          }
      
          private class VoucherOrderHandler implements Runnable {
      
              @Override
              public void run() {
                  String queueName = "stream.orders";
                  while (true) {
                      try {
                          //1.获取消息队列中的订单信息
                          List<MapRecord<String, Object, Object>> list = stringRedisTemplate.opsForStream().read(
                                  Consumer.from("g1", "c1"),
                                  StreamReadOptions.empty().count(1).block(Duration.ofSeconds(2)),
                                  StreamOffset.create(queueName, ReadOffset.lastConsumed())
                          );
                          //2.判断消息获取是否成功
                          if (list == null || list.isEmpty()) {
                              //2.1 如果消息获取失败，说明没有消息，继续进行下一次循环
                              continue;
                          }
                          //2.2 如果消息获取成功，对消息进行处理
                          // 解析list中的订单信息
                          MapRecord<String, Object, Object> record = list.get(0);
                          Map<Object, Object> map = record.getValue();
                          VoucherOrder voucherOrder = BeanUtil.fillBeanWithMap(map, new VoucherOrder(), true);
                          //对消息进行处理,即创建订单
                          handleVoucherOrder(voucherOrder);
                          //3. ACK确认
                          stringRedisTemplate.opsForStream().acknowledge(queueName, "g1", record.getId());
                      } catch (Exception e) {
                          log.error("订单消息处理异常！", e);
                          //4.消息处理异常，说明存在被读取但未被ack的订单消息
                          handlePendingList();
                      }
                  }
              }
          }
      ```

    

    



# 面试相关问题

## 1. 项目里为什么要用消息队列：



## 2. 请求很多，消息堆积处理不过来了如何应对

## 3. 用户在消息堆积时以为卡了，多次请求怎么处理

## 4. 项目都有哪些表

##  5. 超卖问题怎么解决？

## 5. 秒杀场景下扣减库存太慢了怎么解决

## 7. Redis大key怎么解决？

## 8. 什么是热Key

## 9. 如何解决热key问题

## 10.  短信登录的短信怎么发送的

## 11. 项目的拦截器详细讲讲

## 12. 怎么保存验证码

## 13.项目里存在redis里的key的格式，存的什么

## 14. 如何标识用户

## 15.项目的权限刷新什么意思

## 16. 旁路缓存机制具体解决的什么场景

## 17 更新缓存失败了怎么办

## 重试的时候，缓存中的错数据被访问多次了，怎么解决

## 项目为什么要加个消息队列

## 抢优惠券没有及时处理怎么办

## 抢优惠券处理完了如何通知用户

## Redis的Zset

## Zset的范围查询的时间复杂度是多少





