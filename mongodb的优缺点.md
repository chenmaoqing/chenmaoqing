#  mongodb的优缺点 #

对比mysql, mongo的优缺点有：

### 缺点 ###

-	不支持事务操作

-	占用空间过大

-	MongoDB没有如MySQL那样成熟的维护工具

-	无法进行关联表查询，不适用于关系多的数据

-	复杂聚合操作通过mapreduce创建，速度慢

-	模式自由，  自由灵活的文件存储格式带来的数据错误

-	预分配模式带来的磁盘瓶颈。

mongodb采用数据文件预分配模式来生成数据文件，**数据文件的大小从64M开始**，**每增加一个文件，大小翻倍，直到2G，**
以后每次增加数据就会生成2G左右的数据文件，结合mongodb的mmap内存模型，对于写数据文件，将随机写转换为顺序写，
一定程度上缓解了磁盘的io压力。

但在实际使用中，遇到了在预分配2G的数据文件时**，如果磁盘io较慢，则mongodb基本锁死，无法响应请求的情况。**
持续时间则根据磁盘io的性能来确定。这个问题在2.0之后版本可能会有些改善，但在磁盘性能低的服务器上，该问题依旧存在.

这个问题目前没有太好的解决方案，只能建议使用读写性能比较好的服务器来跑mongodb。

**在数据存量大于内存大小时，mongodb遇到冷数据查询速度变慢。**

mongodb使用mmap的内存管理模式，如果查询的都是热数据，那么会在内存中直接查询，如果遇到冷数据，就需要从磁盘读取，
并将一部分热数据从内存卸载掉.

有人曾经说mongodb内存管理是加载固定大小的文件块到内存，即如果冷数据在磁盘上，他会根据请求的数据，
加载一定大小的数据块到内存，并卸载掉同样的热数据，这个操作本身会带来一定io.

因为**mongodb使用的是全局锁，在某个操作缓慢时，整个操作队列会全部变慢**。

这个问题造成了mongodb会出现偶发性堵塞问题，连带整个库的性能下降。

 
该问题在应用需要尽量避免出现，需要将mongodb的数据大小规划好，尽量不要使数据量超过内存的大小，如果超过内存大小后，尽量不要去请求冷数据。

### Mongodb全局锁机制。 ###

mongodb最大的问题或者可以说是它的锁机制，在2.2版本之前，一个实例只有一个读写锁，不管有多少数据库和数据集合，
当一个操作进行时其他操作只能等待，在2.2版本后，mongodb锁降低了粒度，改为按库锁。

MongoDB 使用的是“readers-writer”锁， 可以支持并发但有很大的局限性，当一个读锁存在,许多读操作可以使用这把锁，
然而, 当一个写锁的存在，一个单一的写操作会exclusively 持有该锁，同时其它读，写操作不能使用共享这个锁；举个例子，假设一个集合里有 10 个文档，多个 update 操作不能并发在这个集合上，即使是更新不同的文档。

**删除数据集合后空间不会自动释放**

**mongodb删除集合后磁盘空间不释放，只有用db.repairDatabase()去修复才能释放。**

**修复可能要花费很长的时间,在使用db.repairDatabase()去修复时一定要停掉读写，并且mongodb要有备机才可以，不然千万不要随便使用db.repairDatabase()来修复数据库，切记。**

但是在修复的过程中如果出现了非正常的mongodb的挂掉，再次启动时启动不了的，需要先修复才可以，
可以利用./mongod --repair --dbpath=/data/mongo/ 
如果你是把数据库单独的放在一个文件夹中指定dbpath时就指向要修复的数据库就可以。



###replica set一些隐含问题 ###

   -	replica set模式最多支持12台服务器，而有投票权的服务器只支持7台，如果超过7台服务器，需设置部分服务器为无投票权服务器

   -	replica set模式中，一个set服务器如果小于2台服务器，则自动故障恢复不会起作用，如果4台服务器出现2/2互相ping不通的情况，同样不会自动故障恢复。一般来说，一个set中尽量是有单数服务器。

   -	replica set中，因为mongodb是按照时间进行操作，如果set中某个服务器时间超前或者延迟，很容易出现secondaries不断尝试更新oplog或者同步延迟的问题。甚至造成某些操作失败，如drop操作。

###分片模式的一些隐含问题 ###

   -	config server尽量按照官方的要求，有3个configserver，如果只有2个configserver，则shard的自动负载均衡和自动切片功能不可用。

   -	api中的nearest模式在shard中，判断的是set到mongos的距离而非set到client的距离，在切片模式下，尽量不要使用nearest模式，可能会造成一些请求延迟增加的问题。

## 优点 ##

-	 文档结构的存储方式，能够更便捷的获取数据

-	 内置GridFS，支持大容量的存储

-	 内置Sharding，分片简单

-	 海量数据下，性能优越

-	 支持自动故障恢复（复制集）

 

mongodb是一个介于nosql数据库和mysql数据库之间的一个数据存储系统，它没有严格的数据格式，但同时支持复杂查询，而且自带sharding模式和Replica Set模式，支持分片模式，复制模式，自动故障处理，自动故障转移，自动扩容，全内容索引，动态查询等功能。扩展性和功能都比较强大。

mongodb在数据查询方面，支持类sql查询，可以一个key多value内容，可以组合多个value内容来查询，支持索引，支持联合索引，支持复杂查询 ，支持排序，基本上除了join和事务类型的操作外，mongodb支持所有mysql支持的查询，甚至某个客户端api支持直接使用sql语句查询mongodb。

mongodb的sharding功能目前日渐完善，支持自定义范围分片，hash自动分片等，分片自动扩容，shard之间自动负载均衡等功能。实际使用中功能还不错。