# Redis

## 上下文

- Redis学习参考：146-Redis核心技术与实战
- 通过这个排查问题：![Redis问题图像图](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9tbWJpei5xcGljLmNuL21tYml6X2pwZy9IWGlhYnV0Mm9pY25NeU15T1hxTk1CbnNmUGxrM0tOUmtHdGlhM0pzY1pPQldrNUN5MzZ1VlhVSUYyQ3V3NXFwUzhacVgwdDhMVmpIUHhTbDZ6RkhZN2RTZy82NDA?x-oss-process=image/format,png)

## 使用场景与解决的问题

Redis 的特点就是利用 C语言和内存数据库 这样的特性提供存储服务。

1. 数据库
2. 旁路缓存，作为后端MySQL数据库的上级缓存
3. 秒杀
4. 消息队列
5. 分布式锁

## 基本架构、原理和生态

Redis 整体包括如下模块：
1. 网络模块接受客户端请求。
2. 控制模块负责数据的 CRUD 工作，还有数据的管理。
3. 存储模块主要是 AOF日志 和 RDB快照，保证数据不会丢失。
4. 高可用控制模块：主从复制、哨兵、分片。

Redis 网络模块使用的是多线程/多路复用IO，对于网络连接来说，阻塞点一个是 等待客户端建立连接，一个是 等待客户端发送数据，而这个两个地方可以用多个套接字，异步执行，一有通知就让主线程执行。

Redis 支持很多数据结构，包括可以自定义数据结构，其中关键的是：
1. 全局Hash表，利用内存的特性，达到O1的速度，表Value并非原始数据，而是指针。HashMap的扩容也是渐进式rehash，当操作一个Item的时候，附带把这个Item附近的邻居一起迁移到新表，保证阻塞成本在大量操作中分担散开。
2. 简单动态字符串：这个要注意结构体本身带了很多元数据，如果是字符串长度短，可能得不偿失。
3. 压缩列表：在数量比较少的情况下（可以设置阈值），用列表代替Hash甚至其他的，是一种速度空间取舍，因为量小，O1和On近似，注意就是，压缩列表有头尾指针还有长度，这三个是O1的速度。
4. 跳表：用二分法，多级定位Item。

On 在体量大的数据里很昂贵的，所以Redis提供了很多批量获取数据的命令，比如 SCAN，具体参见：https://redis.io/commands。

集合类型的数据要注意 Bibkey，因为一旦数据体量大了，各方面操作不留神就会导致阻塞。

AOF 在主线程执行，是写后日志（保证语句正确和能够成功执行），直接在某一次事务的最后记录命令到磁盘，其刷盘策略有三个，同步、每秒、有系统决定，越同步性能越低，根据业务场景判断。AOF文件大了之后，Redis会执行AOF重写，本质就是合并写入语句。AOF重写是后台线程去完成的，这要注意，1、Fork线程有成本，可能会阻塞；2、AOF的内存数据需要拷贝新的一份，注意；3、对于新数据，放到缓冲区，最后再合并。

RDB 是快照，用于主从传输和定期备份，不用 AOF 是因为AOF重放慢而且大小比较大。执行快照的时候，使用COW写时复制，所以要注意，如果数据写比例大的，内存的使用要计算在内，如果是 BigKey 的话，COW甚至会导致阻塞。因为快照的时机没办法确定，所以使用 AOF/RDB 混合的方式，一段时间RDB周期内 + 几个AOF落盘。

主从 的设置是为了应对 读多写少 的场景，从库 清空数据库从主库、获取到 RDB，然后重放。如果从库很多，可能主库的网络压力会很多，所以可以把一部分从库认定某个从库作为主库也可以。在运行过程中，如果主从同步遇到网络不佳，新数据会写入一个 repl_backlog_buffer 环形缓冲区，可一定超过了缓冲区大小，日后重新连接会直接执行全量同步，这时候要注意主库的内存和网络压力。

哨兵机制/集群 是为了应对主从故障切换的，主要关注的问题就是：1、主观/客观判断节点下线；2、使用哨兵集群保证高可用；3、当前集群内网络压力。4、集群面临的共同问题。

分片集群 本质上是拆分业务来分担单个集群负载的方法。主要关注的问题：1、如何划分/拆分，Hash槽？2、节点的新增问题。第一个问题，Redis的做法是用Hash槽，对Key进行计算，然后选定节点，（可以用一些手段人为把某些Key放到同一个节点上，因为范围查询和事务不支持跨节点）。第二个问题，Redis并无使用注册中心这样的服务存储集群数据，利用相互通信各自持有的方式来组织，这种设计符合实际情况。因为Redis的节点数量是有限的（官方上限1000），而引入注册中心也是引入更大的复杂度，所以这种 相互沟通 + 客户端缓存 + 重定向 的方式比较适合。

Redis 刚开始没有集群，所以有了 Codis集群，其使用 Zookeeper 管理维护数据，使用 Proxy 管理路由表，在数据迁移的时候支持异步（bigkey集合数据按集合元素为单位迁移），客户端也不需要调整，单机和集群相同，出来时间比较早，生产环境比较成熟。所以根据自己的情况去选择。

Pika 针对SSD设计的键值数据库，充分利用 SSD 的特点：1、用SSD作为保存更大容量的数据，内存作为一个大缓存。2、使用 binlog 代替 AOF/RDB，省去了很多 AOF/RDB 的性能成本。3、兼容Redis接口。4、因为使用 SSD，可以使用多线程，性能更高一点。总之，缺点就是，毕竟使用SSD作为支持存储，性能会比Redis低一些，本质上还是权衡。

NVM 的优势就是 持久化内存、容量大、速度与DRAM接近。有两种模式，一个是 Memory模式， 大内存+速度快+不能持久化，一个是 App Direct 模式，持久化+速度慢点。有了持久化后，就不需要AOF和RDB，但速度慢点，也是取舍。

6.0新特性：
1. 多线程处理网络，本质上就是进一步细分阻塞点，用多线程去处理网络请求。
2. 服务端协助客户端缓存，本质上就是利用客户端的缓存，服务端的作用就是通知读取到的缓存是否过期。
3. ACL访问控制，可以指定用户和其允许使用的命令。

## 应用

### 旁路缓存

如果需要对写请求进行加速，我们选择读写缓存；如果写请求很少，或者是只需要提升读请求的响应速度的话，我们选择只读缓存。

缓存容量的设计一般是 15%-30%，兼顾访问性能和内存空间开销。

缓存不一致，对于只读缓存，两种操作逻辑：
1. 先删数据库后删缓存，导致其他请求读到就值。
2. 先删缓存再删数据库，导致其他请求迅速从数据库取出旧值，所以延迟双删即可。
对于读写缓存，建议写回，如果是异步，我建议操作数据库交给消息队列，拿到ACK就返回。

缓存遇到问题：
1. 缓存雪崩：大量的Keys同时过期，或者业务量突然大，Redis无法承受，或者Redis本身有节点挂掉了。方案：1. 服务熔断和降级限流；2、对过期的Keys进行微调时间；3、提前预防。
2. 缓存击穿：某个热点数据突然过期了，大量请求给了数据库。这个时候建议不用设置过期时间。
3. 缓存穿透：访问不存在的值，导致压力给到了数据库。方案：前端检查过滤、布隆过滤器、设置新Key然后提供默认值。
4. 缓存污染：缓存污染：有大量不再访问的数据留在缓存中，每隔一段时间只请求一次，后面就不请求了。LFU 在 LRU 的基础上加入了对访问次数的统计。LRU 策略更加关注数据的时效性，而 LFU 策略更加关注数据的访问频次，一般可以将 lfu_log_factor 取值为 10。

关于LRU：
Redis 对LRU的实现并没有使用链表，而是为每个数据维护一个时间戳，在淘汰数据的时候，先随机取出一批数据，然后把最小的数据淘汰掉。`CONFIG SET maxmemory-samples 100` 设置取出多少个。

### 消息队列

Redis 有基于 List 的方案，提供 Blocking阻塞，也有基于 Streams 的解决方案。

### 分布式锁/原子操作

原子操作：要么是单个命令（INCR/DECR 命令），要么就是Lua脚本。

Redis 分布式锁这个，其实可以自己实现，本质上就是 原子操作 + 信号量，注意的点就是：1、谁加锁谁解锁，识别客户端ID；2、可以使用ListBlock去阻塞，代替信号量；3、加过期时间，万一解锁失败或者事务失败。集群的话，Redlock 这个东西，原理就是多少个节点确认就算加锁成功。

### 事务

基本原理：之前不是 单命令/Lua脚本 可以保证原子性，所以可以在客户端层面开启一个工作队列，然后添加命令，最后提交去统一执行，相当于内部生成 Lua 脚本，保证原子性。

问题就是，如果在构造 工作队列 的时候挂掉了，那没事，顶多清空就可以回滚，如果是最后提交执行的时候挂掉了，如果开启AOF，还可以去除掉，如何没有就没办法了，无法保证原子性。

在构造 工作队列 的时候，如果需要修改的值修改了，怎么保证隔离性？用 WATCH 命令，如果检测的值发生了变化，事务也就取消了。提交执行的时候，因为单线程，隔离性可以得到保证。

至于持久性基本上靠AOF/RDB保证，而且落盘的时机设置。

### 秒杀

秒杀场景对支持系统的要求：
1. 瞬态的并发访问量高
2. 读多写少，读操作都是简单的查询操作（库存）

【 Redis做秒杀的话，主要就是锁定库存，用 INCR/DECR 命令，把用户ID写到消息队列作为安全保底，然后查询，这个时候后端数据流使用 SingleFlight 这种机制。 】


## 内部细节

如果集合数据体量小，Redis提供了一个权衡版本的底层数据结构：压缩列表：
1. hash-max-ziplist-entries：表示用压缩列表保存时哈希集合中的最大元素个数。
2. hash-max-ziplist-value：表示用压缩列表保存时哈希集合中单个元素的最大长度。
看用压缩列表存储图片ID的例子。


数据管理优化：
1. 惰性删除：对于Del的Key，标记删除，然后定期删除。
2. 过期Key延迟删除：当客户端读取Key的时候，检查是否过期，如果过期才执行删除。
3. 删除大体量数据使用 ASYNC 选项，还有 UNLINK 命令（批量异步删除Keys）。
4. 对于LRU淘汰规则，随机选择在一批里小步走淘汰，避免形成阻塞。

可能导致阻塞的地方：
1. 对集合进行全量查询，聚合操作，Bigkey。
2. 删除大量的Key，Redis为了高效管理内存，需要为内存做一些插入链表的工作，这个会阻塞，尤其是清空数据库。
3. AOF的日志同步写，加载 RDB 文件的过程。
4. CPU：单一架构需要绑核，多CPU的话注意不要总线串门。17 | 为什么CPU结构也会影响Redis的性能？

分片集群节点的增删虽然会导致重新Hash，但是这个过程阻塞很小，一方面渐进式迁移，另方面哈希槽信息本身不大。

出现延迟：
1. 慢查询。1、用其他高效的命令代替；2、聚合操作放在客户端去完成；3. 最好少遍历操作。
2. 可能有很多过期的Keys正在删除，所以不要设置大量同时过期的Keys。
3. AOF重写会大量占用磁盘IO，虽然fsync是后台完成，但是如果上一个fsync还没有完成，就会导致阻塞。
4. 内存swap。
5. 内存大页。内存大页本质上是取舍，系统在 COW写时复制的时候，会拷贝大页，从而提升内存使用。
6. 最好控制实例数据体量2～4GB，所以考虑使用 32位的。
7. 再看看是不是受到其他程序影响，比如监控程序。
8. 可以打开 no-appendfsync-on-rewrite yes 表示AOF重写的时候，不进行落盘操作。或者使用 SSD。

Redis 键值对的 key 可以设置过期时间。默认情况下，Redis 每 100 毫秒会删除一些过期 key，具体的算法如下：
1. 采样 ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP 个数的 key，并将其中过期的 key 全部删除；
2. 如果超过 25% 的 key 过期了，则重复删除的过程，直到过期 key 的比例降至 25% 以下。

删除数据后，内存会出现碎片，查看命令 INFO memory，mem_fragmentation_ratio 超过 1.5 （碎片率超过50%）就需要采取措施了，首先打开 config set activedefrag yes，然后：
1. active-defrag-ignore-bytes 100mb：表示内存碎片的字节数达到 100MB 时，开始清理；
2. active-defrag-threshold-lower 10：表示内存碎片空间占操作系统分配给 Redis 的总空间比例达到 10% 时，开始清理。
3. active-defrag-cycle-min 25： 表示自动清理过程所用 CPU 时间的比例不低于 25%，保证清理能正常开展；
4. active-defrag-cycle-max 75：表示自动清理过程所用 CPU 时间的比例不高于 75%，一旦超过，就停止清理，从而避免在清理时，大量的内存拷贝阻塞 Redis，导致响应延迟升高。


redis 跟客户端或者节点交互的时候会设置缓冲区，分布叫做 输入缓冲区、输出缓冲区，对于 输入缓冲区，溢出的原因两种：1、写入 bigkey，一下子写入多个百万级别的集合数据；2、redis处理过慢，阻塞，客户端请求比较频繁，请求命令堆积。

`CLIENT LIST` 命令查看：
cmd，表示客户端最新执行的命令。这个例子中执行的是 CLIENT 命令。
qbuf，表示输入缓冲区已经使用的大小。这个例子中的 CLIENT 命令已使用了 26 字节大小的缓冲区。
qbuf-free，表示输入缓冲区尚未使用的大小。这个例子中的 CLIENT 命令还可以使用 32742 字节的缓冲区。qbuf 和 qbuf-free 的总和就是，Redis 服务器端当前为已连接的这个客户端分配的缓冲区总大小。这个例子中总共分配了 26 + 32742 = 32768 字节，也就是 32KB 的缓冲区。

输入缓冲区并不可以调大，所以只能从命令处理速度着手。

输出缓冲区溢出原因：
1. 服务器端返回 bigkey 的大量结果；内存拷贝。
2. 执行了 MONITOR 命令；MONITOR 会持续监测，占用缓冲区越来越多，所以主要用在调试环境中，不要在线上生产环境中持续使用 MONITOR。
3. 缓冲区大小设置得不合理。

默认缓冲区设置 `client-output-buffer-limit normal 0 0 0`，normal 表示当前设置的是普通客户端，第 1 个 0 设置的是缓冲区大小限制，第 2 个 0 和第 3 个 0 分别表示缓冲区持续写入量限制和持续写入时间限制。0 代表不限制，当订阅频道的信息很多的时候，会占用很多的输出缓冲区，所以需要限制。`client-output-buffer-limit pubsub 8mb 2mb 60` pubsub 参数表示当前是对订阅客户端进行设置；8mb 表示输出缓冲区的大小上限为 8MB，一旦实际占用的缓冲区大小要超过 8MB，服务器端就会直接关闭客户端的连接；2mb 和 60 表示，如果连续 60 秒内对输出缓冲区的写入量超过 2MB 的话，服务器端也会关闭客户端连接。

主从数据全量同步的时候使用 `复制缓冲区`，从节点连接主节点，本质上也是客户端，一旦缓冲区溢出，就会被破关闭连接，同步失败，所以经验：
1. 控制主节点保存数据大小，一般2-4GB，这样让速度更快些。
2. client-output-buffer-limit 设置大小，
3. 从节点数量过多也会导致缓冲区占用的内存很大，所以要控制节点数量。

主从数据增量同步使用 `复制积压缓冲区`，就是 repl_backlog_buffer，数据结构是环形结构，调整 repl_backlog_size 参数即可。


主从数据不一致，是因为异步执行的，这个时候可以保证网络要顺畅，还有就是可以开发外部检测工具，用 `INFO replication` 来检测同步情况，针对差的，通知客户端移除掉。

拿到过期的数据，是因为删除策略，一个是惰性删除，就是等请求了才去检查是否过期，一个是定期删除。定期删除就是Redis防止阻塞不会一次性删除大量数据，惰性删除就是，在3.2版本之前不会判断从库是否过期，因为写操作由主库负责，并且通知下发，还有一种情况就是，有些设置过期命令仅仅是在XX时间之后删除，同步到从库的时候，这个时候实际删除时间会延迟主库。

不合理配置：
1. protected-mode 要关闭，尤其是哨兵的时候。
2. cluster-node-timeout 最好 10-20秒。
3. lave-serve-stale-data 设置为no，这样从库不回处理写命令了，一致性。

### 脑裂

客户端往两个节点上写数据。

原因：
1. 主库还未同步数据到新添加的从库就挂了，从库升级为主库，数据丢失。这种情况 比较 master_repl_offset 和 slave_repl_offset 差值就可以判断。
2. 主从切换的时候，有一个客户端跟旧主库通信，但是这个旧主库因为网络问题，被判断失效了。就在新主库要求从库同步数据的时候，旧主库这段时间的新数据就被清空了。

Redis 已经提供了两个配置项来限制主库的请求处理：
- min-slaves-to-write：这个配置项设置了主库能进行数据同步的最少从库数量；
- min-slaves-max-lag：这个配置项设置了主从库间进行数据复制时，从库给主库发送 ACK 消息的最大延迟（以秒为单位）。
这两个配置项组合后的要求是，主库连接的从库中至少有 N 个从库，和主库进行数据复制时的 ACK 消息延迟不能超过 T 秒，否则，主库就不会再接收客户端的请求了。这样一来，原主库就会被限制接收客户端请求，客户端也就不能在原主库中写入新数据了。

### 数据分布优化，数据倾斜

第一种，数据量倾斜，原因 比如 bigkey 占据了整个节点，还有 Slot 手动分配不均衡，还有 Hash Tag。

Hash Tag 的作用是手动制定某些数据在同一个节点上，这样在集群下，做事务、数据范围查询 才方便。

第二种，因为热点导致的数据访问倾斜，方法就是指定多个副本，Key的名称随机化处理。

### 集群优化

Redis 官方给出了 Redis Cluster 的规模上限，就是一个集群运行 1000 个实例。

Redis集群没有中心化注册中心，节点之间通过互通交换信息，加上定期的Ping检查，而这个必然节点间的网络压力很大。可以把 cluster-node-timeout 调大，这样可以减少节点之间相互之间检测的频率，设置时间时，最好先测一下：

```sh
tcpdump host 192.168.10.3 port 16379 -i 网卡名 -w /tmp/r1.cap
```

## 运维

先确定到底是不是变慢了，有没有针对当前硬件的基线性能。方法：redis-cli 命令提供了–intrinsic-latency 。然后用 iPerf 测量网络延迟。

### 工具

1. INFO命令

2. 面向 Prometheus 的 Redis-exporter 监控

3. 数据迁移工具 Redis-shake

4. 集群管理工具 CacheCloud


## 参考

- 146-Redis核心技术与实战：加餐（六）| Redis的使用规范小建议
