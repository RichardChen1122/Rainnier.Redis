# Redis初识
## Redis 特性
- 速度快  
    - 10W OPS  
    - Redis 数据存在内存里  
    - 单线程  
-  持久化  
   - Redis 所有数据再内存中，对数据的更新将异步保存在磁盘上  
   - RDB AOF 两种持久化方式  
-  多种数据结构  
    - 字符串、哈希表、链表、sets、 sorted sets  
        **下面这两个本质是数据库**  
        位图(布隆过滤器)  
        HyperLogLog: 超小内存（12k）唯一值计数，上亿数据也只有12k  
        **下面这两个本质是集合**  
        GEO：地理信息定位，算经度纬度
- 支持多种语言  
- 功能丰富
  - 发布订阅
  - Lua脚本
  - 支持事务
  - pipeline
- “简单”
  - 核心代码只有两万多行
  - 不依赖外部库
  - 单线程模型
- 复制
  - 主从复制
- 高可用性和分布式
  - Redis-Sentinel（2.8）支持高可用
  - Redis-Cluster（3.0）支持分布式

## Redis 典型使用场景
  - 缓存系统
  - 计数器，微博转发评论量的计数
  - 消息队列系统
  - 排行榜
  - 社交网络
  - 实时系统 垃圾邮件处理系统，消息队列的缓冲，用位图来实现布隆过滤器

## Redis 启动
- Redis 
- 三种启动方式
  - 最简启动
  - 配置文件启动
  - 动态参数启动

## Redis 常用配置
- daemonize => 是否守护进程
- port => 端口号
- logfile Redis 系统日志
- dir => 工作目录


# Redis 进阶
## Redis API 的使用和理解
- 通用命令
  - ```keys```
    - 一般不再生产环境使用， 复杂度为o(n)
    - 热备的从节点
    - scan
  - ```dbsize```
    -  计算key 的总数
    -  可以用在生产环境， 时间复杂度o(1)
  - ```exists key```
    - 判断key 是否存在， 1=>存在， 0 =>不存在
    - 一般情况下可以随便使用，o(1) 复杂度
  - ```del key [key ...]```
    - 删除key, o(1) 复杂度
  - ```expire key second```
    - 设置过期时间 单位为秒
    - ```ttl key``` => 查看key 的剩余过期时间
      - 返回-2 --> 没有这个key
      - 返回-1 --> 没有设置过期时间
    - o(1) 复杂度
    - ```persist key``` => 去掉key 的过期时间
  - ```type key```
    - 返回值有 string， hash， list， set， zset， none（在不存在的key上执行）
    - o(1) 复杂度
  - ```info memory```
- 数据结构和内部编码
- 单线程架构

## 数据结构和内部编码
- redisObject
  - 数据类型 type
    - string hash set zset list
  - 编码方式 encoding
    - aw int ziplist linkedlist hashmap intset
  - 数据指针
  - 虚拟内存

## 单线程
  - 单线程为什么这么快？
    - 纯内存, 响应时间快
    - 非阻塞IO
    - 避免线程切换和竞态消耗
  - 注意
    - 一次只运行一条命令
    - 拒绝长命令  keys flushall flushdb， slow lua script， mutil/exec, operate big value
    - 其实不是纯单线程
      - fysnc file descriptor
      - close file descriptor

## 字符串
- 键值结构
    | key     | value                      |
    | ------- | -------------------------- |
    | hello   | world                      |
    | counter | 1                          |
    | bits    | [1,0,1,1,0,1]              |
    | json    | {"product":{"id":""value}} |
- 场景
  - 缓存
  - 分布式锁
  - 计数器
  - 等等等等
- 命令
  - ```get key```
  - ```set key value #不管key是否存在，都设置```
  - ```setnx key value #必须key不存在才设置(add)```    
  - ```set key value xx #必须key存在才设置(update)```
  - ```mset key1 key2 key3```
  - ```mget key1 key2 key3```
  - ```del key```
  - ```incr key```
  - ```decr key```
  - ```incrby key i```
  - ```decrby key i```
  - ```getset key newvalue #set key newvalue 并返回旧value(先get旧，再set新)```
  - ```append key value #string.append```
  - ```strlen key #获取value长度(byte)```
  - ```incrbyfloat key 3.5 #增加对应的浮点值```
  - ```getrange key start end #获取字符串制定下标所有的值(包含start、end)```
  - ```setrange key index value #set字符串制定下标的值```
- 实战value
  - 实现个人主页的访问量
    ```incr userid:pageView```
  - 缓存视频的基本信息(数据源在MySQL中)
    ```
    public VideoInfo get(long id){
        string redisKey=redisPrefix+id;
        VideoInfo info = redis.get(redisKey);
        if(info==null){
            info = mysql.select(id);
            if(info!=null){
                redis.set(redisKey, serialize(info));
            }
        }
        return info;
    }
    ```
- 实现分布式的自增ID，多个应用
  - incr id

## 哈希
- 结构
<table>
   <tr>
      <th>key</th>
      <th>value</th>
   </tr>
   <tr>
        <th>user:1</th>
        <td>
            <table>
                <tr>
                    <td>name</td>
                    <td>richard</td>
                </tr>
                <tr>
                    <td>email</td>
                    <td>123@abc.com</td>
                </tr>
                <tr>
                    <td>id</td>
                    <td>12</td>
                </tr>
            </table>
        <td>
   </tr>
   <tr>
        <th>user:2</th>
        <td>
            <table>
                <tr>
                    <td>name</td>
                    <td>nash</td>
                </tr>
                <tr>
                    <td>email</td>
                    <td>1223@nash.com</td>
                </tr>
                <tr>
                    <td>id</td>
                    <td>13</td>
                </tr>
            </table>
        <td>
   </tr>
</table>

- 特点
  - field 不能相同, value 可以相等
  - small redis

- 命令
  - ```hget key field       #获取hash key 对应field的value```
  - ```hset key field value #设置hash key 对应field 的value```
  - ```hdel key field       #删除```
  - ```hexists key field    #判断hash key 的 field 是否存在```
  - ```hlen key             #hash key 的field 数量  ```
  - ```hmget key field1 field2 ... fieldN```
  - ```hmset key field1 value1 field2 value2 ... fieldN valueN```
  - ```hgetall key          #返回hash key 对应所有的field 和value，这个要小心使用```
  - ```hvals key            #返回hash key 对应所有field 的value```
  - ```hkeys key            #返回hash key 对应所有field```
  - ```hset ```
  - ```hsetnx```
  - ```hincrby```
  - ```hincrbyfloat```
  
- 实战
  - 记录网站每个用户个人主页的访问量
    ```hincrby user1 pageview count```
  - 缓存视频的基本信息
    ```
        public VideoInfo get(long id){
            string redisKey = redisPrefix + id；
            Map<string,string> map = redis.hgetAll(redisKey);
            VedioInfo info = transferMapToVedioInfo(map);
            if(info==null){
                info = mysql.get(id);
                if(info!=null){
                    redis.hmset(redisKey,transferVideoInfoToMap(info));
                }
            }

            return info;
        }
    ```

## 列表
-  列表结构
    | key            | ELEMENTS       |
    | -------------- | -------------- |
    | user:1:message | a - b -c -d -e |
    - 有序
    - 可重复
    - 可左右弹出

- 命令
  - ```rpush key value1 value2 ... valueN        #从列表右端插入值(1~N个)```
  - ```lpush key value1 value2 ... valueN        #从列表左端插入值(1~N个)```
  - ```linsert key before|after value newvalue   #在list指定的值前|后插入newValue 复杂度o(n)```
  - ```lpop key                                  #从列表左侧弹出一个item```
  - ```lpop key                                  #从列表右侧弹出一个item```
  - ```lrem key count value```
  - ```ltrim key start end```
  - ```lrange key start end(包括)                #查列表的index 范围```
  - ```lindex key index                          #查列表指定位置的item```
  - ```llen key                                  #获取列表长度```
  - ```lset key index newValue                   #set列表指定位置的值```
  - ```blpop key timeout                         #lpop 的阻塞版本列表指定位置的值```
  - ```brpop key timeout                         #阻塞的等待pop item```

- 实战
  - 微博的时间轴 timeline
- Tips
  - lpush + lpop =  Stack  
  - lpush + rpop = queue
  - lpush + ltrim = Capped Collection
  - lpush + brpop = message queue

## set集合
- 结构
  - key -- values
  - 

- 特点
  - 无序
  - 集合不允许插入重复数据 
  - 可以集合内和集合间操作

- 集合内命令API
  - ```sadd key element                   #向集合key 添加element 如果存在则失败```
  - ```srem key element                   #删除 ```
  - ```scard                              #集合大小```
  - ```sismember key                      #是否在集合内```
  - ```srandmember key count              #随机获取集合内count个元素 ```
  - ```spop key                           #随机弹出一个元素 ```
  - ```smembers key                       #获取集合中的所有元素 返回结果无序，小心使用```
  - ```sscan```

- 实战
  - 抽奖系统
    - spop 的随机性
  - 微博的Like、踩、赞
  - 标签Tag
    - sadd user:1:tags tag1 tag2 tag5
    - sadd user:2:tags tag2 tag3 tag5 tag7
    - 给标签添加用户
      - sadd tag1:users user:1 user:3

- 集合间API
  - ```sdiff setKey1 setKey2      #两集合差集```
  - ```sinter setKey1 setKey2     #两集合交集```
  - ```sunion setKey1 setKey2     #两集合并集```
  - ```sdiff|sinter|sunion store destkey```

- 实战
  - 社交网站的共同关注

- Tips
  - sadd 做些标签方便的应用
  - spop/srandmember 做些关于随机方面的应用，如抽奖
  - sadd + sinter 可以做些社交关系的应用

## zset 有序集合
- 结构
    key   ---  score(代表顺序)  --- value

- 有序集合 VS 集合
  - 相同点
    - 无重复元素
  - 不同
    - 有序无序
    - 有序集合api 时间复杂度偏高

- API
    - ```zadd key score element           #score可重复，element不可重复 o(logN)```
    - ```zrem key elemnt1 ... elementN```
    - ```zscore key elemnt                #返回元素的score```
    - ```zincrby key increScore element   #对元素score 增加increScore 值```
    - ```zcard key                        #返回元素个数```
    - ```zrank key                        #获取元素的排名 从小到大```
    - ```zrevrank key                        #获取元素的排名 从大到小```
    - ```zrange key start end [withscores]#返回排序区间内的元素 o(log(N)+m)```
    - ```zrevrange key start end [withscores]#返回排序区间内的元素 o(log(N)+m) 从高到低```
    - ```zrangebyscore key minScore maxScore  [withscores]#返回分数区间内的元素 o(log(N)+m)```
    - ```zcount key minScore maxScore     #返回分数区间内的元素个数 o(log(N)+m)```
    - ```zremrangebyrank key start end    #删除排名区间内的元素 o(log(N)+m)```
    - ```zremrangebyscore key minScore maxScore    #删除分数区间内的元素 o(log(N)+m)```
    - ```zinterstore```
    - ```zunionstore```
- 实战
  - 排行榜

# Redis 客户端
## Java客户端 Jedis
- 获取
  - maven dependency

- 连接
  - Jedis 直连
    - TCP 连接 socket协议
    ```
        Jedis jedis = new Jedis("127.0.0.1", 6379);
        jedis.set("hello","world");
    ```
- Jedis 连接池
  - Jedis 直连
    - 基于TCP连接
  - Jedis 连接池
    | 方式   | 优点                               | 缺点                                                           |
    | ------ | ---------------------------------- | -------------------------------------------------------------- |
    | 直连   | 简单方便，适用于少量长期连接的场景 | 每次新建关闭TCP，Jedis 对象线程不安全                          |
    | 连接池 | 预生成，降低开销                   | 相对于直连 使用相对麻烦，需要参数保证，参数设不好 依旧会出问题 |
  - 使用示例
    ```
        Jedis jedis =null;
        try{
            jedis = jedisPool.getResource();
            jedis.set("hello", world);
        }
        catch(Exception e){
            e.printStackTrace();
        }
        finally{
            if(jedis!=null){
                jedis.close()  //归还至连接池
            }
        }
    ```

# 瑞士军刀Redis
## 慢查询
- 生命周期
  - Redis 生命周期: client发送命令-->Redis排队-->Redis 执行命令-->返回结果给client
  - 慢查询发生在第三阶段，指真的是执行命令慢(如 ```keys *```, ```hgetall```)，而不是因为网络或排队的因素。客户端超时不一定是慢查询，但慢查询是客户端超时的一个可能性因素
- 两个配置
  - slowlog-max-len
    - 先进先出队列
    - 固定长度
    - 保存在内存中
  - slowlog-log-slower-than
    - 慢查询阈值
    - 单位：微秒
    - 超过设定值就划为慢查询
- 配置方法
  - 默认值 
    - config get slowlog-max-len = 128
    - config get slowlog-log-slower-than = 10000
  - 修改配置文件，重启
    - 不建议，第一次配置可以
  - 动态配置
    - config set slowlog-max-len value
    - config set slowlog-log-slower-than value
- 慢查询命令
  - ```slowlog get [n]  #获取慢查询队列```
  - ```slowlog len      #获取慢查询队列长度```
  - ```slowlog reset    #清空```
- 运维经验
  - slowlog-log-slower-than 不要设置过大，默认10ms， 通常1ms
  - slowlog-max-len 不要设置过小，通常设置1000左右
  - 理解命令生命周期
  - 定期持久化慢查询，持久化到mySql，便于查询历史记录，帮助分析问题

## Pipeline流水线
- 什么是流水线
  - 一次模型: 1次时间 = 1次网络时间 + 1次命令时间
  - 批量模型: n次时间 = n次网络时间 + n次命令时间
  - 流水线： 1次pipeline(n条命令) = 1次网络时间 + n次命令时间
  - 两点注意：1.Redis 的命令时间很快是微秒级别的 2. pipeline 每次条数要控制(网络)
- Pipeline Jedis 实现
  - 没有Pipeline
  ```
    Jedis jedis = new Jedis("127.0.0.1",6379);
    for(int i=0;i<10000;i++){
        jedis.hset("hashkey:" + i, "field" +i, "value" +i);
    }
  ```
  - 使用pipeline
  ```
    Jedis jedis = new Jedis("127.0.0.1",6379);
    for(int i=0;i<100;i++){
        Pipeline pipeline = jedis.pipelined();
        for(int j=i*100;j<(i+1)*100;j++){
            pipeline.hset("hashkey:" + i, "field" +i, "value" +i);
        }
        pipeline.syncAndReturnAll();
    }
  ```

  - 与原生M操作对比
    - M操作时原子的
    - pipeline 的结果是非原子的
  - 使用建议
    - 注意每次pipeline 携带的数据量
    - pipeline 每次只能作用在一个Redis 节点上
    - M操作和pipeline 的区别

## 发布订阅
- 角色
  - 发布者
  - 订阅者
  - 频道
- 模型
  - Redis Server 存在一个频道，比如羽球TV频道
  - 发布者发布消息hello到这个频道
  - 订阅了这个频道的每个订阅者就会收到hello这个消息
  - 一个订阅者可以订阅多个频道
  - 无法消息堆积，新订阅者无法获得历史发布的消息
- API
  - publish channel message
    ```publish badminton:tv "hello world"```
  - subscribe channel1 ... channelN
  - unsubscribe
  - psubscribe pattern
  - punsubscribe pattern
  - pubsub channels  #列出至少有一个订阅者的频道
  - pubsub numsub channel #列出给定频道的订阅者数量
  - pubsub numpat #列出被订阅模式的数量
- 发布订阅VS消息队列
  - 消息队列是所有消息的订阅者竞争队列
  - 考虑好使用尝尽
- 总结
  - 发布订阅模式中的角色
  - 重要的API
  - 消息队列和发布订阅的使用场景

## bitmap位图
- 位图
  - 本质上是string
  - 只存储 0 和1
  - "big" 的二进制 01100010|01101001|01100111
    ```
        set hello big
        getbit hello 0     => return 0
        getbit hello 1     => return 1
    ```
    Redis 可以直接操作位  

- API
  - ```setbit key offset value        #给位图的指定索引设置值```
  - ```getbit key offset              #获取位图的指定索引的值```
  - ```bitcount key [start end]       #获取位图指定范围位值位1的个数， start end 单位是字节```
  - ```bitop op destkey key1 ... keyN #做一个bitmap 的 and or not xor 操作并保存结果在destKey 中 ```

- 使用经验
  - 最大512MB
  - 注意setbit 时的偏移量，可能较大的耗时
  - bitmap不一定好，要根据情况使用

## HyperLogLog
- 数据结构
  - 基于HyperLogLog 算法：极小的空间完成独立用户统计
  - 本质还是字符串

- 三个命令
  - ```pfadd key element [element..]```
  - ```pfcount key [key ...] 计算hyperloglog 的独立总数```
  - ```pfmerge destkey sourcekey [sourcekey ...] 合并多个hyperloglog```
- Demo
  ```
    pfadd 2020_03_06:unique:ids "uuid-1" "uuid-2" "uuid-3" "uuid-4"
    pfcount 2020_03_06:unique:ids   #返回4
    pfadd 2020_03_06:unique:ids "uuid-1" "uuid-2" "uuid-3" "uuid-99" #添加结果1
    pfcount 2020_03_06:unique:ids   #返回5
  ```
- 内存消耗
  - 百万独立用户 1天消耗内存15k

- 使用经验
  - 是否能容忍错误？(错误率0.81%)
  - 是否需要单条数据？hyperloglog 无法获得单条数据
  - 是否需要很少的内存去统计

## GEO
- GEO 是什么
  - 存储经纬度，计算两地距离，范围计算等
  - 可以根据两地的经纬度，算两地距离
  - 算出指定范围内的目标
- 5个城市经纬度
    | 城市   | 经度   | 纬度  | 简称         |
    | ------ | ------ | ----- | ------------ |
    | 北京   | 116.28 | 39.55 | beijing      |
    | 天津   | 117.12 | 39.08 | tianjin      |
    | 石家庄 | 114.29 | 38.02 | shijiazhuang |
    | 唐山   | 118.01 | 39.28 | tangshan     |
    | 保定   | 115.29 | 38.51 | baoding      |
- 命令
  - ```geoadd key longitude latitude member       #增加地理位置信息```
  - ```geopos key member ... memberN       #获取地理位置信息```
  - ```geodis key member1 member2 [unit]       #计算两地距离，m(米) mi(英里)```
  - ```georadius key longitude latituede radiusm|km|ft|mi [witcoord] [withdis] [withhash] [Count count] [asc|desc] [store key] [storedist key]       #获取指定位置范围内的地理位置信息集合```
    withcoord:返回结果中包含经纬度  
    withdist:返回结果中包含距离中心位置的距离  
    withhash:返回结果中包含geohash  
    COUNT count:指定返回结果的数量  
    asc|desc: 返回结果按照距离中心节点的距离做升序或者降序  
    store key: 将返回结果的地理位置信息保存到指定key  
    storedist key: 将返回结果距离中心节点的距离保存到指定key  

- 相关说明
  - since 3.2+
  - type 是zset
  - 没有删除API，删除用的是zset 的zrem key member

# Redis持久化
## 持久化的作用
- 什么是持久化
  - redis 所有数据都在内存中，对数据的更新将异步地保存到磁盘上
- 持久化方式
  - 快照
    - MySQL Dump
    - Redis RDB
  - 写日志
    - MySQL Binlog
    - Hbase HLog
    - Redis AOF

## RDB
- 什么是RDB
  - redis 创建RDB文件，内存生成快照生成二进制文件存在硬盘中
  - 重启后可以从磁盘中的RDB文件重新载入内存，恢复redis
  - 作为主从复制的复制媒介

- 触发机制-主要三种方式
  - save(同步)
    - 执行save 命令，生成RDB文件
    - 在save 期间，对redis 的所有其他操作会被阻塞，如果数据很大，就很慢
    - 文件策略: 如果存在老的RDB文件，新的会替换老的
    - 复杂度：O(N)
  - bgsave(异步)
    - 执行bgsave命令，会使用linux 的fork()函数生成Redis 的子进程，让子进程去生成RDB文件，完成后告诉主进程RDB生成成功
    - fork()函数依然会阻塞Redis, 但是它很快，但是在一些情况下依然会出现问题
    - Redis 依然可以响应客户端其他命令，不阻塞
    - 复杂度：O(N)
    - 消耗额外内存
  - 自动生成RDB
    - save 900s 1changes
    - save 300s 10changes
    - save 60s 10000changes
    - 满足上述3个条件任意一个自动生成RDB文件
    - RDB不一定很好，频率无法很好的控制
  - 默认配置
    - save 900 1
    - save 200 10
    - save 60 10000
    - dbfilename dump.rdb
    - dir ./
    - stop-writes-on-bgsave-error yes
    - rdbcompression yes
    - rdbchecksum yes
  - 最佳配置
    - 关闭自动save
    - dbfilename dumpe-${port}.rdb
    - dir /bigdiskpath  #选择比较大的硬盘，按照端口做硬盘分盘
    - stop-writes-on-bgsave-error yes
    - rdbcompression yes
    - rdbchecksum yes

- 触发机制-不容忽略的方式
  - 全量复制
  - debug reload
  - shutdown

## AOF
- RDB 现存的问题
  - 耗时、耗性能
    - O(n)的复杂度，耗时
    - fork()消耗内存，copy-on-write策略
    - DISK I/O：IO性能
  - 数据丢失、不可控
    - 上次保存后又写入新数据，但是突然宕机，数据就丢失了

- 什么是AOF
  - 一个写命令就在日志里面增加一条记录
  - 发生宕机后，就按照AOF文件完整恢复

- AOF的三种策略
  - always
    - redis 写命令会刷新缓冲区，会使用linux 每条命令都会 fsync 到硬盘
    - 不丢失数据，但IO开销大
  - everysec （一般用这个）
    - redis 写命令会刷新缓冲区，每秒都会把缓冲区 fsync 到硬盘
    - 允许只丢一秒数据，适当保护磁盘，但可能会丢失1s数据
  - no
    - redis 写命令会刷新缓冲区，根据操作系统决定何时 fsync 到硬盘
    - 不用管，不可控

- AOF重写
  ```
    set hello world
    set hello java
    set hello hehe
  ```
  - 要关注的就是最后条数据hehe， AOF 重写 set hello hehe
  - 过期数据，AOF重写就不考虑
  - 减少磁盘占有量
  - 加快速度恢复

- AOF 重写的两种实现方式
  - bgrewriteaof
    - 类似bgsave，会异步执行，会fork 一个子进程，做AOF重写
  - AOF 重写配置
    - auto-aof-rewrite-min-size AOF文件重写需要的尺寸
    - auto-aof-rewrite-percentage AOF文件的增长率
  - AOF 统计项
    - aof_current_size aof当前尺寸，单位字节
    - aof_base_size AOF上次启动和重写的尺寸 单位字节
  - 同时满足下面两个条件，可以做AOF重写
    -  aof_current_size > auto-aof-rewrite-min-size
    -  (aof_current_size - aof_base_size)/aof_base_size > auto-aof-rewrite-percentage

- 推荐配置
  - appendonly yes
  - appendonlyfilename "appendonly-${port}.aof"
  - appendfsync everysec
  - dir /bigdiskpath
  - no-appendfsync-on-rewrite yes #AOF重写很消耗性能
  - auto-aof-rewrite-percentage 100
  - auto-aof-rewrite-min-size 64mb
  
## RDB和AOF的抉择
- RDB 和AOF 比较
  - 启动优先级： RDB低 AOF高
  - 体积：RDB小(有压缩) AOF大
  - 恢复速度： RDB快 AOF 慢
  - 数据安全性 RDB容易丢数据，AOF根据策略决定
  - 轻重 RDB重 aof 轻
  
- RDB的"最佳策略"
  - "关" 主从复制的时候不能关
  - 集中管理，数据集中备份是个不错的选择
  - 主从，从开？根据实际需求

- AOF 最佳策略
  - "开"，如果只做缓存那可以关
  - AOF重写集中管理
  - everysec

- 最佳策略
  - 小分片，每个redis max memory 为4G，各种开销小
  - 根据缓存或者存储
  - 监控硬盘、内存、负载、网络
  - 足够的内存，不要100%内存部满

# Redis开发运维常见问题
## fork 操作
- 同步操作
  - 大部分情况下速度快，但是有时也会有问题
  - 与内存量息息相关：内存越大，耗时越长
  - info：latest_fork_usec  上次fork 执行的微秒数

- 优先使用物理机或者高效支持fork 操作的虚拟化技术
- 控制Redis 实例最大可用内存：maxmemory
- 合理配置Linux 内存分配策略vm.overcommit_memory=1 (默认为0)
- 降低fork频率：例如放宽AOF重写自动触发时机，不必要的全量复制
## 子进程开销和优化
- CPU开销 
  - 开销：RDB和AOF 文件生成，属于CPU密集
  - 优化：不要做CPU绑定，不和CPU密集型应用一起部署
- 内存
  - 开销：fork 内存开销，copy-on-write(linux写时复制机制)，父子进程共享相同物理内存页，但父进程有写请求会创建一个副本，如果父进程有大量写入，子进程开销就很大
  - 优化： echo never > /sys/kernel/mm/transparent_hugpage/enabled
- 硬盘的消耗
  - 开销：AOF和RDB文件写入，可以结合iostat, iotop 分析
  - 不要和高硬盘负载服务部署在一起
  - no-appendfsync-on-rewrite=yes
  - 根据写入量决定磁盘类型，如ssd
  - 单机多实例持久化文件目录可以考虑分盘

## AOF追加阻塞
- 通常是每秒刷磁盘策略
- 主线程会监测对比上次fsync 的时间，如果大于2s就阻塞，小于2s就通过
- 主线程不能阻塞
- AOF阻塞定位
  - Redis 日志
  - redis命令 info persistence

# Redis复制原理和优化
## 什么是主从复制
- 单机有什么问题
  - 机器发生故障
  - 容量瓶颈，16G内存， redis 用了12G， 如果要64G内存怎么办？
  - QPS瓶颈，几百万QPS怎么办
- 主从复制的作用
  - master-slave
    - slave 复制master
  - 一主多从
    - 数据可以有多个副本，更加高可用
    - 扩展读性能,读写分离，master 有很多读写，把读的流量分流，负载均衡
 - 简单总结
   - 一个master可以有多个slave
   - 一个slave只有有一个master
   - 每个slave 也可以有slave
   - 数据流向时单向的，master 到slave

## 复制的配置
- 两种方式
  - slaveof命令
    - 6380作为6379 的从节点：
      redis-6380> slaveof 127.0.0.1 6379
      #这个命令时异步的，会直接返回，但实际需要时间
      redis-6380> slaveof no one  #取消复制
  - 配置
    - slaveof ip port
    - slave-read-only yes
  - 比较
    - 命令无需重启，但不便于管理
    - 配置可以同一配置，但需要重启redis
- 主从复制配置操作
  - info replication -> role:master
## 全量复制和部分复制
- RunId 和偏移量
  - RunId时redis的唯一标识，重启会变，salve感受到master变了他会做全量复制
  - 偏移量，一致的时候，主从同步。
    - info replication -> master_repl_offset
- 全量复制过程
  - slave给master 发 psync(runId, offet) -> 初始psync(？, -1)
  - master FULLRESYNC 告诉slave runId 和我的偏移量
  - slave保存master基本信息，master 会去执行bgsave 保存RDB快照
  - 在bgsave 期间新的写入会进入repl_back_buffer这个缓冲区
  - master send RDB to slave
  - master send buffer to slave
  - slave flush old data
  - load new RDB and buffer
- 全量复制开销(很大)
  - bgsave 时间
  - RDB文件网络传输时间
  - 从节点清空数据时间
  - 从节点加载RDB时间
  - 可能的AOF重写
- 部分复制的过程
  - 假如slave 与master 断开连接
  - 断开期间新的写入会进入repl_back_buffer这个缓冲区
  - 网络恢复后重连，slave 发送psync {runID} {offset}
  - 如果offset在buffer区间内，master 会响应continue，并把这部分数据发给slave
## 故障处理
- 故障不可避免，不要害怕故障
  - 无自动故障转移，就很崩溃
  - 自动故障转移，保证服务整体的可用性
- 主从结构的故障
  - slave宕机
    - 使用这个slave客户端转到使用其他slave
  - master宕机
    - 只读客户端正常
    - 客户端去找一个master->其他slave 转移到这个master
  - sentinel可以主从复制故障转移
## 开发运维的常见问题
- 读写分离
  - 读流量分摊到从节点
  - 可能遇到的问题
    - 复制数据延迟
    - 读到过期的数据，slave 节点不能删除数据，salve 可能会读到脏数据
    - 从节点故障，怎么做客户端分离

- 主从复制不一致
  - 例如maxmemory不一致：丢失数据
  - 数据结构优化参数(hash-max-ziplist-entries)：内存不一致

- 规避全量复制
  - 第一次全量复制 
    - 第一次不可避免
    - 小主节点，低峰， maxmemory 不要设置太大，晚上负载低的时候做这个事
  - 节点运行ID不匹配
    - 主节点重启(运行ID变化)
    - 故障转移，例如哨兵或集群
  - 复制积压缓冲区不足
    - 是个队列，大小为1M就只能buffer 1M
    - 网络中断，部分复制无法满足
    - 增大复制缓冲区配置 rel_backlog_size，网络"增强"

- 规避复制风暴
  - 单节点复制风暴
    - 问题：主节点，多从节点复制
    - 解决：更换复制拓扑，但这种也会有问题
  - 单机器复制风暴
    - 机器宕机后，大量全复制
    - 解决：主节点分散多机器
    - master挂了Slave 晋升成master

# sentinel
## 主从复制高可用？

## 架构说明

## 安装配置

## 客户端连接

## 实现原理

## 常见开发运维问题