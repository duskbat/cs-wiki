
# 3. Redis

### 分布式缓存技术选型
Redis Memcached  
| 缓存      | 数据类型 | 持久化 | 集群       | 过期策略          |
| --------- | -------- | ------ | ---------- | ----------------- |
| redis     | 5种      | 支持   | 原生支持   | 惰性删除+定期删除 |
| memcached | 1种      | 不支持 | 原生不支持 | 惰性删除          |

- redis 支持的数据类型更丰富
- redis 支持数据持久化, 可以保存在硬盘, 重启的时候能再次加载
- memcached 没有原生的集群模式, redis原生支持集群
- memcached 过期只用惰性删除, redis支持惰性删除和定期删除


### 数据结构  
字符串String、字典Hash、列表List、集合Set、有序集合SortedSet
且每种类型都有本地方法

#### 1 String
动态字符串，预分配冗余空间的方式来减少内存的频繁分配。小于1M直接加倍，超过1M再加1M，字符串最大512M。
Redis 的 SDS(simple dynamic string) API 是安全的，不会造成缓冲区溢出。
```
struct sdsshdr{
  int len;
  int free;
  char[] buf;
}

string 长度
键值对  
先验新增  
批量键值对  
过期和多久过期
计数  
```
bitmap:
setbit mykey {offset} {value(1/0)}



#### 2 list列表
双向链表，插入删除操作快，索引定位O(n)  
当列表弹出最后一个元素之后，该数据结构自动被删除，内存被回收。  
通常当作异步队列使用，将需要延后处理的任务结构体序列化成字符串塞进列表，另一个线程从这个列表中轮询数据进行处理。
```
-   队列
-   栈
-   通过索引查（慢操作）
-   trim（慢操作）  
```
quicklist 
早期版本双向链表和ziplist配合使用，但是指针占用空间大，后期改成将ziplist加上前后指针，节省存储空间。  

#### 3 hash(字典)
底层用哈希表实现.
rehash是渐进式的，会保留新旧两个hash结构，同时查询，然后在后续的定时任务以及hash的子指令中，渐渐将旧hash转到新hash。  
hash可以对结构单独存储，不必像字符串一样全部存取。

#### 4 set
keyset 所有val==null, 可用于去重

#### 5 zset(sorted set)
跳跃列表 以一定概率(官方晋升率25%)选择代表形成上一层(最高32层), 最底层是双向链表, 上层都是单向链表
value+score(权重)


### 文件-事件处理器
Redis 基于 Reactor 模式开发了自己的网络事件处理器：这个处理器被称为文件事件处理器（file event handler）。
redis采用epoll来实现I/O多路复用（linux本身的内核技术），将连接信息和事件放到队列中，依次放到文件事件分派器，事件分派器将事件发送给事件处理器
文件事件处理器使用 I/O 多路复用（multiplexing）程序来同时监听多个套接字(Socket)，并根据套接字目前执行的任务来为套接字关联不同的事件处理器。

当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关 闭（close）等操作时，与操作相对应的文件事件就会产生，这时文件事件分派器就会调用套接字之前关联好的事件处理器来处理这些事件。

虽然文件事件处理器以单线程方式运行，但通过使用 I/O 多路复用程序来监听多个套接字，文件事件处理器既实现了高性能的网络通信模型，又可以很好地与 Redis 服务器中其他同样以单线程方式运行的模块进行对接，这保持了 Redis 内部单线程设计的简单性。

### 单线程
例如在秒杀场景, 落在DB层一定是串行的, 而redis单线程的设计恰如其分
4.0版本在**异步删除大对象**加入了多线程
6.0为了提高**网络IO读写性能**(性能瓶颈在内存和网络,而不是CPU,这是不使用多线程的主要原因)

### 过期策略
通过过期字典(字典key指向具体的数据key, val是毫秒精度的时间戳long long类型的整数)保存数据过期时间. 定时遍历删除+惰性删除
- 定时扫描策略
  **每秒10次**过期扫描, 贪心策略:随机选20个, 如果过期比例超过1/4, 重复; 为防止线程卡死，每次不超过25ms  
  这样会导致卡顿, 不要让大批key同时过期
- 惰性删除
  访问时检查


### 内存淘汰机制
1. volatile-lru（least recently used）：从已设置过期时间的数据集（server.db[i].expires）中挑选最少使用的数据淘汰
2. volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
3. volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. allkeys-lru（least recently used）：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）
5. allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
6. no-eviction：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。  

**4.0 版本后增加以下两种**：  
1. volatile-lfu（least frequently used）：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
2. allkeys-lfu（least frequently used）：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

### 持久化
通常是从节点进行持久化，从节点作为备份节点，没有客户端请求的压力
1. 快照持久化（snapshotting，RDB）
   默认方式, 通过创建快照,获取数据某个时间点的副本, 可以复制到从服务器或者留存到本地 //通过COW的方式
   一段时间内超过一定数量的key发生变化，会创建快照
   Redis.conf 
   ```
   save 900 1           #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
   save 300 10          #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
   save 60 10000        #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发BGSAVE命令创建快照。
   ```
2. 只追加文件（append-only file, AOF）
   开启AOF后执行的每一条更改命令, redis都会将该命令写入硬盘里的AOF文件. AOF文件和RDB文件位置相同, 都是通过dir参数设置的, 默认的文件名是append only.aof
   **fsync**: 控制刷写硬盘
   appendfsync everysec  #每秒执行一次fsync操作
   appendfsync always    #每次有数据修改发生时都会写入AOF文件,这样会严重降低Redis的速度. 能保证完整性
   appendfsync no        #让操作系统决定何时进行同步 交给内核
3. redis 4.0开始支持 RDB和AOF的混合持久化(默认关闭, 可通过配置项 aof-use-rdb-preamble 开启)


### 热key
https://dongzl.github.io/2021/01/14/03-Redis-Hot-Key/index.html
#### 热key带来的问题
流量集中，达到服务器处理上限
IO打满，其他key受影响
落在某一个单实例，无法通过扩容解决
redis不可用导致缓存击穿，DB也会被打爆
#### 如何发现
客户端侧，服务器节点侧，redis命令
- 业务经验预估
- 客户端收集验证
  对客户端工具（SDK，如Jedis、Redisson）进行封装，在发送请求前进行收集，同时定时把收集到的数据上报到统一的服务进行聚合计算。
- 代理层收集
  跟上面的客户端收集类似
- redis命令
  - hotkeys
  - monitor
- 节点抓包
#### 解决方案
水平扩容，增加实例副本数量
二级缓存，本地缓存
热key备份

### redis序列化反序列化
org.springframework.data.redis.serializer.RedisSerializer
对于json而言, jackson 是官方实现 fastjson也有相应实现

### 分布式锁使用场景
进程之间保证有序性，分布式服务下需要顺序执行
### 分布式锁
使用 setnx(set if not exists) 指令，处理完成后del。但是如果出现异常会出现死锁，那么需要加expire, 但是expire不是原子指令。2.8版本官方加了set指令的扩展参数。
#### 超时问题
分布式锁不能解决超时问题。一个更安全的方案是给set指令的value参数加一个随机数，释放锁的时候先匹配随机数，然后删除释放锁。但是匹配value和删除操作不是原子操作。可以用lua脚本处理。
#### 可重入性
使用threadLocal存储当前持有锁的计数。
#### 哨兵集群中，主节点挂掉，锁失效问题
Redlock算法，加锁时向过半的节点发送指令，过半的节点成功则认为加锁成功，释放锁的时候向所有节点发送del指令。
使用时需要做很多的额外细节处理，运维，出错重试，时钟漂移等，不建议使用


### redis事务
Redis 可以通过 **MULTI，EXEC，DISCARD** 和 **WATCH** 等命令来实现事务功能。
DISCARD 丢弃 MULTI 到 DISCARD 之间的所有指令, 必须在 EXEC 之前使用.
WATCH 必须先于 MULTI 使用. 乐观锁的方式, 盯住一些变量, 在执行的时候检查是否被修改, 如果被修改返回错误, 客户端处理(通常是重试)
redis事务不支持rollback, 所以不满足原子性.
可以理解为是多个命令顺序打包执行, 且执行过程不会被中断(即便前面执行失败也会继续执行后续指令).

### 缓存穿透
缓存中和DB中都没有

### 缓存击穿  
缓存中没有, 没有起到缓存的效果, 全打到DB上了;  
可以在缓存之后加一层布隆过滤器;

### 布隆过滤器  
用一个BitMap, 使用n个不同的散列函数将一个元素映射到n个位点上  
一个元素如果判断结果为存在的时候元素不一定存在，但是判断结果为不存在的时候则一定不存在。  
可以添加元素，但不能删除元素  
在使用bloom filter时，绕不过的两点是预估数据量n以及期望的误判率fpp，  
在实现bloom filter时，绕不过的两点就是hash函数的选取以及bit数组的大小。
    
### 缓存雪崩
缓存在同一时间大面积失效, 失去缓存效果.
**针对redis服务不可用的情况**:
用redis集群, 避免单机宕机导致整个缓存服务不可用; 限流, 避免同时处理大量请求;
**针对热点缓存失效的情况**:
设置缓存失效时间添加随机值;

### 缓存与DB一致性 TODO
更新还是淘汰：
- 更新操作在业务逻辑复杂的时候开销大，淘汰的代价只是一次miss;
- 更新操作需要串行化，否则有可能引起不一致

首先改db和淘汰缓存不是原子的，那么就有先后，考虑到先更新db后淘汰缓存出现失败的情况，那么一定要先淘汰缓存；
再考虑到更改db过程中有可能读到缓存中，那就可以采用延时双删，异步双删


如下三种模式胡扯
旁路缓存，读写穿透，异步缓存写入
三种模式:
1. Cache Aside Pattern（旁路缓存模式）
   **失效**：先从cache取数据，没取到则从数据库种取数据，成功后放到缓存中。
   **命中**：cache中取到，返回
   **更新**：先存到数据库中，然后删除缓存
   
   写操作频繁会导致缓存数据频繁删除, 影响命中率. 
   解决:
   - 强一致场景, 将DB和缓存加锁后同时更新
   - 弱一致场景, 同样同时更新, 不过可以不加锁, 此时如果更新DB失败, 会造成不一致, 应当缩减缓存过期时间
2. Read/Write Through Pattern（读写穿透）
   将更新操作交由缓存层自己代理
   写: 
   - cache 中不存在，直接更新 DB。
   - cache 中存在，则先更新 cache，然后 cache 服务自己更新 DB（同步更新 cache 和 DB）。
   
   读:
   - 从 cache 中读取数据，读取到就直接返回.
   - 读取不到，由cache服务从 DB 加载，写入到 cache 后返回响应。
   
3. Write Behind Pattern（异步缓存写入）
   跟读写穿透类似, 不同点是异步批量更新DB. 类似于操作系统刷写磁盘操作


### 分布式解决方案
CAP中AP满足
#### 主从同步
增量同步 + 快照同步
主备同步
启动初始化
rewrite
#### 哨兵
哨兵集群，客户端先访问哨兵，哨兵给出节点地址；节点不可用则由哨兵选出下一个主节点
主从复制是异步复制，那么主节点故障时可能会丢失消息；解决方案是必须至少有x个从节点y秒内有心跳，否则不可用
#### 压力过大
分片集群 代理集群
#### AKF
扩展问题，可以分为三个维度扩展
X:主主 主从 主备
Y:业务、技术服务维度拆分
Z:数据维度拆分
分片怎么做: redis做自有的治理, 请求到redis上之后redis通过算法决定哪个槽位处理

### copy-on-write(COW)
如果有多个调用者同时请求相同资源（如内存或磁盘上的数据存储），他们会共同获取相同的指针指向相同的资源，直到某个调用者试图修改资源的内容时，系统才会真正复制一份专用副本（private copy）给该调用者，而其他调用者所见到的最初的资源仍然保持不变。


### BIO NIO 问题
> socket -> bind -> listen

**历史时期BIO**:
因为client通过TCP连接kernel(内核), 每个连接就是一个文件描述符(如fd8), 导致线程从kernel读取fd8的时候产生阻塞, 必须通过另外的线程等待连接. 线程的开销大
**历史时期NIO**:
通过循环读取fd, 系统调用会有用户态到内核态切换的过程, client多的时候循环次数多, 时间开销大
**历史时期多路复用**:
内核改变, 调用select(), 内核去遍历. 但是每次调用select会传递所有的文件描述符, 并且每次遍历所有的文件描述符
**epoll**:
I/O event notification facility
epoll_create, epoll_ctl, epoll_


### 硬盘吞吐
可达百兆/G 
内存 ns级别 硬盘 寻址时间ms级别
总线标准通常两种(消费级): SATA PCIe
SSD的规格协议通常两种(消费级): AHCI NVMe (与上文对应

### 死锁
- 超时回滚
- 等待图：通过锁的信息链表和事务等待链表