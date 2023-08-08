# MongoDB

## MongoDB高手课程 PPT

[入门](./geektime-mongodb-course-chapter-1.pdf)

[熟练到精通](./geektime-mongodb-course-chapter-2pdf)

[分片集群和运维](./geektime-mongodb-course-chapter-3.pdf)

[架构师进阶](./geektime-mongodb-course-chapter-4.pdf)

> https://github.com/geektime-geekbang/geektime-mongodb-course 

## 复制集

复制集 类似于 主从，服务于 高可用，常见复制集是 一主二从 三节点部署。

数据通过 oplog 节点间复制。

故障恢复：具有投票权的节点互相发送心跳，超过5次未收到心跳判定失联，如果失联的是主节点，选举，如果从节点就不需要，复制集最多可以有 50 个节点，投票权最多7个节点。

影响选举：大多数节点存活，被选举的主节点，要有较新的oplog，人为配置的优先级。

M 保证数据不丢失：写入的时候可以设置几个从节点成功写入才返回ACK，否则哪怕主节点挂掉，也会判断事务失败。

## 分片集

M 是原生支持的。

分片主要就是如何分片：范围、Hash、Zone区域

多少分片，三个指标最大值：

1. 按照容量来，总容量 和 单机容量。
2. 按照内存大小来。
3. 按照并发数来。

范围划分注意：

集群 Cluster > 分片 Shard > 块 Chunk > 文档 Doc > 片健Shard Key

片健 可以理解为 主键索引，要 分布均匀、定向性好、基数大。

扩容：监控达到60%，考虑扩容，因为迁移需要很长时间。

> PPT 3

## 监控

监控的指标要具备一定的容错 和 留出足够的处理时间。

> 分片集群的监控参阅 PPT 3.4 MongoDB 监控最佳实践

## 备份和恢复

M 的备份两种：延迟节点备份 和 全量备份 + Oplog增量。

延迟节点备份：设定某个节点延迟处理 oplog，然后在那个节点上备份，要保证延迟从节点 和 oplog开始时间大于48小时。

全量备份方式：

1. 复制数据文件

	必须关闭节点做，或者  db.fsyncLock()  锁节点，不要忘记解锁，然后在从节点做，不能影响投票，然后整个过程是宕机一个节点。

2. 文件系统快照

	利用文件系统的快照功能，不用停机，要保证 数据和journal必须在同一个卷上。

3. mongodump

	最灵活，但速度上也是最慢的。但是 mongodump 出来的数据不能表示某个个时间点，只是某个时间段。所以要做好幂等。

> PPT 3.5 MongoDB 备份与恢复 和 3.6 备份和恢复操作

## 安全

> PPT 3.7 MongoDB 安全架构 和 3.8 MongoDB 安全加固实践

### 认证和鉴权

用户认证：
1. 用户名密码
2 x509证书
3. LDAP外部认证（企业版）
4. Kerberos外部认证（企业版）

节点认证：
1. Keyfile（将统一 Keyfile 文件拷贝到不同的节点 Keyfile 就是一个字符串）
2. X.509（不同节点使用不同的证书）

M 使用基于角色的鉴权，有内置，也可以自定义。

> 内置角色 参阅 PPT 3.7 MongoDB 安全架构 - MongoDB 内置角色及权限继承关系

### 加密

传输加密：MongoDB 支持 TLS/SSL 来加密 MongoDB 的所有网络传输(客户端应用和服务器端之间，内部复制集之间)。

落盘加密：M 支持对数据库加密。

字段级加密：M支持 单独文档字段通过自身密钥加密，不需要自己开发。

### 审计

对 DDL、DML、用户认证 进行记录，便于调查。

## M 全家桶

mongod 数据库软件

mongo 命令行工具

mongos 路由进程，分片环境下用

mongodump/mongorestore 备份和恢复工具

mongoexport/mongoimport CSV/JSON 导入导出工具，主要迁移数据用

Compass GUI管理工具

Ops Manager 企业版，集群管理工具，支持 K8S。

BI Connector 企业版，SQL解释器

MongoDB Charts 企业版，可视化软件

Atlas 付费/免费 云托管服务

## 文档模型设计

### 基础

先根据业务需求得到模型，列出模型之间的关系，使用 内嵌方式 进行建模。

一对一关系，直接内嵌，如果最后超过 16MB，就需要拆分。

一对多的关系，用数组内嵌，如果体量超过 16MB，或数组大小上限未知，建议拆分出来，用 ID 来。

### 工况细化

先确定业务是如何使用数据的，然后进行优化。比如 读写操作比例，数据量大小，查询模式、查询参数，数据写入模式。

对于上限不明确的数组信息，单独提取出来，用 ID数组 来引用，查询的时候使用聚合 $lookup 来还愿当初设计的模型。

** 这种引用方式，一般用于 文档太大（数组无上限）和 数组元素需要频繁修改。**

活数据和死数据也要分开。

注意：
1. M 对对象之间的引用外建需要自己保证。
2. $lookup 模仿关联查询，只支持 left outer join。
3. $lookup 的关联目标不能是分片表。

> 《100-MongoDB高手课 16》
> $lookup 使用  https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/#std-label-unwind-example


### 设计模式 

分桶设计模式：像日志采集业务，因为体量过亿，而在M中，每条日志都有一定量的元数据，为了节约存储，可以 按批 存储，一个文档代表一个桶，包含100条数据。

列转行设计模式：一文档的字段频繁变动，那就通过 { "列名称": "列值" } 对象数据的形式存储。

版本号字段：M没有版本号管理，所以自己维护。

按批写入：因为写入很频繁，尤其是统计的时候，所以可以攒个100条数据一起发送。比如统计次数，10次发一次，结果 +10 即可。

渐进式聚合计算：做一些聚合操作，或者统计操作，操作的体量非常大，这个时候可以维护一个结果，在每一次添加数据的时候，自己去加1，相当于把聚合操作的成本平摊到每次操作上。

### 事务

持久性保证：journal & Replication

一致性保证：readConcern & writeConcern

隔离性保证：readConcern

---

M 通过 “多少从节点成功写入” 来保证数据丢失，对于 事务 来说，如果从节点没有写入成功 就挂掉了，事务在客户端可以回滚。

{w: "majority", j: true}

如果更加严格，j: true，就是必须写入 journal 落盘才ACK。

对于 MySQL 的当前读，M 里直接 Abort 错误。

如果一个事务已经开始修改一个文档，在事务以外尝试修改同一个文档，则事务以 外的修改会等待事务完成才能继续进行

> Transactions and Write Concern https://www.mongodb.com/docs/manual/core/transactions/#std-label-transactions-write-concern 

---

M 的隔离级别通过 readConcern 来实现：

available:读取所有可用的数据

local:读取所有可用且属于当前分片的数据，默认设置

majority 可以有效防止 脏读，读到已提交的。

linearizable 解决主节点失联的情况，配合 maxTimeMS（超时）。用途：如果写入的时候，挂了一个节点，然后根据 majority 可能读到旧数据，所以使用这个可以保证所有节点都写入了，否则返回错误。

snapshot 快照读 相当于 串行化 和 可重复读 （跟 MySQL 不太一样），能解决幻读和不可重复读，因为所有的东西都从快照里读取。—— 会用 snapshot 实现可重复读。

---

默认事务60秒内完成，否则取消。

读的时候出现写入之后马上，这个时候可以选择从主节点读，用 readPreference 参数选择主节点还是从节点，还可以使用 Tag 对节点分组读。

多文档事务中的读操作必须使用主节点读;

readConcern只应该在事务级别设置，不能设置在每次读写操作上。

> PPT 第2.10讲 事务开发:多文档事务

## Change Stream

针对变更的回掉函数。

原理：是基于 oplog 实现的。它在 oplog 上开启一个 tailable cursor 来追踪所有复制集上的变更操作，最终调用应用中定义的回调函数。

> PPT 第2.11讲 Change Stream

## 最佳实践

- 设置连接池的大小

- 设置最大等待时间，杀掉太慢的查询

- writeConcern 使用 majority 保证数据安全

- readConcern 想好了用

- 复制集/分片集 的连接字符串，越多越好，因为能更好的平衡负载。

- 用 mongodb+src:// 提供域名而非IP地址。

- 不要在 M 前面使用负载均衡，因为有很多是节点绑定的，M自己会处理负载均衡。

- 游标：最好把查询条件收紧，尽可能地都用到。

- 查询要有对应的索引，尽量使用 覆盖索引，通过 projection 减少返回客户端的内容。

- 更新尽可能只包含需要更新的内容，也尽量使用批量插入，使用 TTL 自动过期日志类型的数据。

- 文档结构不要用太长的字段名，不要太深的数组嵌套，不要使用中文。

- 分页不要用 Skip + Limit，用 查询条件 + 排序 + Limit，比如 ID 大于上个分页的最大值。

- 在没有索引的时候，避免使用全文档扫描，避免使用 count。

- 如果使用事务，尽可能让涉及事务的文档分布在同一个分片节点上。


## 索引

> PPT 3.9 MongoDB 索引机制(上) 和 3.10 MongoDB 索引机制(下)

---

M 索引背后是 B-树：
1. B树就是B-树
2. B+树在B-树的基础上，为叶子节点增加链表指针。
3. B*树，在B+树的基础上，为非叶子节点增加链表，进一步压缩高度。

MongoDB 索引类型：
- 单键索引
- 组合索引
- 多值索引
- 地理位置索引
- 全文索引
- TTL索引
- 部分索引
- 哈希索引

> 具体使用参考：PPT 3.10 MongoDB 索引机制(下)

组合索引的最佳方式:ESR原则
- 精确(Equal)匹配的字段放最前面：拿到需要的整块。
- 排序(Sort)条件放中间：在块里面，不需要额外的排序。
- 范围(Range)匹配的字段放最后面：因为已经排序，剩下的就是挑选。

【 本质上就是通过技巧避免内存排序。 】

---

索引技巧：
1. 后台创建索引：db.member.createIndex( { city: 1}, {background: true} )
2. 对BI / 报表专用节点单独创建索引


## 性能

> PPT 3.11 MongoDB 性能机制

### 性能瓶颈

应用端：
- 选择访问入口节点

	对于复制集读操作，选择哪个节点是由 readPreference决定的。如果不希望一个远距离节点被选择，要么 将它设置为隐藏节点、通过标签(Tag)控制可选的节点、使用 nearest 方式。

- 等待数据库连接

	总连接数大于允许的最大连接数maxPoolSize，要么 加大最大连接数(不一定有用)，要么优化查询性能。

- 创建连接和完成认证

	设置 minPoolSize(最小连接数)一次性创建足够的连接。避免突发的大量请求。


服务端：
- 排队等待ticket

	ticket相当于令牌桶，限流用的，由 ticket 不足引起的排队等待，问题往往不在 ticket 本身，而在于为什么正在执行 的操作会长时间占用 ticket。优化 CRUD 性能可以减少 ticket 占用时间。
	
	zlib 压缩方式也可能引起 ticket 不足，因为 zlib 算法本身在进行压缩、解压时需要的时 间比较长，从而造成长时间的 ticket 占用。

- 执行请求

	读的流程：加载数据和索引，如果有索引 Ologn，如果没有 On，然后还要排序，要么索引排序，要么内存排序。所以：**不能命中索引的搜索和内存排序是导致性能问题的最主要原因。**
	
	写的流程：首先定位节点和数据，没索引全表扫描，如果用 Journal 写入直接返回，如果没用，到写缓存（写 oplog、写索引、写数据），然后返回。所以写的主要问题在于：**磁盘速度必须比写入速度要快。**

- 合并执行结果

	如果顺序不重要则不要排序。尽可能使用带片键的查询条件以减少参与查询的分片数。


网络：
- 应用/驱动 - mongos，本身的网络质量
- mongos - 片，路由的速度

### 性能工具

> PPT 3.12 性能排查工具

- mongostat: 用于了解 MongoDB 运行状态的工具

- mongotop: 用于了解集合压力状态的工具

- mongod 日志：https://www.mongodb.com/docs/manual/reference/explain-results/ 

- mtools：将所有慢查询 通过图表形式展现。总结出 所有慢查询的模式和出现次数、消耗时 间等。


## 高级集群设计

### 两地三中心

> PPT 3.13 高级集群设计:两地三中心

容灾级别：PPT 3.13 高级集群设计:两地三中心 - 容灾级别

网络层解决方案：本地DNS 或者 GSLB 全局负载均衡 (带DNS功能)。

应用层解决方案：使用负载均衡或虚拟IP，使用同一个Session，同一套数据。

数据库解决方案：基于日志同步，基于文件系统存储镜像同步。

两地三中心：一个地方，两个中心，还有个地方放一个中心作为备份。

### 全球集群

> PPT 3.15 高级集群设计:全球多写

## 上线和升级

> 3.16 MongoDB 上线及升级

1. 性能测试
2. 环境检查：禁用一些系统配置


## 从关系数据库

### 理由

原生支持的分片集群

快速迭代 —— 关系型反范式的问题 

灵活的JSON模式

大数据量需求（TB，PB）✅

多数据中心跨地域部署

地理位置查询

详细参见：《100-MongoDB高手课 44》

### 迁移方案

1. 数据库导出导入

	一次性，需要应用/数据库下线，需要一定时间

2. 批量迁移 ETL（Kettle/Talend）

	本质上对源库进行批量查询，有比较明显的性能影响，不支持实时同步。

3. 实时异构同步 CDC （Informatica/Tapdata）

	本质上是对源库日志文件解析，秒级数据同步，对源库影响小，支持无缝迁移。

4. 应用主导（自动写代码）

	需要开发量，要自己保证数据不丢失。

详细参见：《100-MongoDB高手课 45 和 46》

### Spark + MongoDB

可以用 MongoDB 代替 HDFS，

优势：H是离线，M是实时的，H无索引，M有，H一次写入多次读，M读写混合，

缺点：好像没有？

连接器用 Spark Connector

> 详细参见：《100-MongoDB高手课 47 和 48实战》
> PPT 第2.12讲 MongoDB 开发最佳实践

### SQL

BI Connector，主要是用于数据分析的，SQL 解析器，企业版，开发环境免费，可以使用 MySQL客户端连接，仅支持查询，不支持增删改，复杂的SQL查询，性能上不保证（主要做数据分析的）。

### 容器化部署

节点之间能够互相通讯，必须使用服务名或固定IP地址，要使用持久化存储，要使用专用 MongoDB 的 Ops Manager 进行集群管理（不是 K8S/Openshift）。

MongoDB 支持分片和复制集，可以直接部署在主机上。

### 数据中台

痛点：企业内信息孤岛不能相互打通，生产新业务出来都要重新部署一套新系统支撑性能要求，培训学校不断需要构建新的业务场景，开发低效。

数据仓库和大数据方案：仅仅是把多个部门的数据统一存放到一起，并未打通多个部门，也没有资产管理。

数据中台解决方案：打通信息孤岛，统一资源并且做资产管理，以 API 的形式为部门提供 分析+应用 的能力，可以随时调取，查看。

MongoDB 作为数据平台优势：横向扩展、多种数据模型、高性能高并发、多云跨中心部署。

MongoDB 作为数据平台场景优势：CQRS 模式（理解为读写分离，写用一个模型，读用一个模型），比如 电商下单，数据通过MySQL写入，然后数据流到数据中台（MongoDB）供其他服务掉用，读和写分离了。

详细参见：《100-MongoDB高手课 51 和 52实战》

