## 复制

**旧版复制的实现**

旧版复制效率较低，在Redis2.8之前是使用的这种，2.8之后复用了旧版的复制，并增加了部分重同步的功能，以弥补缺陷。

旧版复制可分为两步：

1. 全量同步

   从服务器发送SYNC命令给主服务器，主服务器执行BGSAVE命令，生成RDB文件，并将RDB发送给从服务器。从服务器收到后载入这个RDB文件。然后主服务器将缓冲区的写命令发送给从服务器。

2. 命令传播

   主服务器将自己执行过的命令发送给从服务器，保证数据一致性。

   


**缺陷**

当出现网络波动、断连的情况，就需要重新开始复制的步骤。

而频繁的BGSAVE对主服务器的性能影响很大，耗费IO、CPU、内存、带宽资源。



**新版复制**

新版复制使用了PSYNC命令来代替SYNC命令。

分为完整重同步和部分重同步，完整重同步与SYNC命令步骤一致。

不同的在于部分重同步。



部分重同步的实现有三个点：

- 主从服务器的复制偏移量
- 服务器的运行ID
- 主服务器的复制缓存区

从服务器复制时，发送PSYNC命令给主服务器，参数是之前复制的主服务器ID和复制偏移量。

`PSYNC <RUNID> <OFFSET>`

- 主服务器收到OFFSET，对比当前的OFFSET对比，如果一致，说明不需要同步。
- 如果不一致，那么看复制缓冲区中是否有OFFSET之后的数据，如果主服务器的复制缓冲区中有OFFSET之后的数据，那么执行部分重同步。否则执行全量重同步。
- 主服务器启动时，生成RUN ID；从服务器同步时，主服务器会把RUN ID 发送给从服务器，从服务器会记录这个RUN ID，之后同步时，会带上这个RUN ID。如果RUN ID与主服务器不一致，那么需要执行全量同步。



从服务器第一次执行时，没有同步过，发送的是 `PSYNC ? -1`，主服务器收到后发送：

`FULLRESYNC <RUN ID> <OFFSET>`。

部分重同步主服务器回复 `CONTINUE`



同步完成后，继续命令传播。



设置复制缓冲区大小 ： `repl-backlog-size`



**心跳检测**

从服务器会默认每秒一次的频率发送命令：

`REPLCONF ACK <OFFSET>`

offset是当前从服务器的偏移量。

心跳检测有三个作用：

1. 检测主服务器的网络连接状态
2. 主服务器检测有无命令丢失，判断是否要重发命令
3. 辅助实现min-slaves



`min-slaves-to-write` 当从服务器小于这个数量，主服务器将拒绝写

`min-slaves-max-lag` 当有`min-slaves-to-write`数量的从服务器延迟大于这个数字，当拒绝写。



## 哨兵

**哨兵与服务器的连接**

哨兵会创建两个连接：

一个是命令连接，用于发送命令，每十秒一次INFO命令，每一秒一次PING命令。

一个是订阅连接，每两秒发送一次信息，包含哨兵自己的信息和主服务器的信息。



**主观下线**

如果某一台哨兵向服务器发送命令，收到无效命令或者没收到命令，那么会标记为此服务器为主管下线。



**客观下线**

当某个哨兵发现服务器主观下线，那么将会向其他哨兵询问是否真的下线了。当其他哨兵也认为是下线了，达到一定数量，那么将会认为是客观下线。



发送 `SENTINEL is-master-down-by-addr host port`给其他哨兵。

客观下线的判断数量可配置：`sentinel monitor master host port num`



**选举**

当判定主服务器为客观下线了，sentinel会进行协商，选出一个领头的哨兵。选举规则是获得大部分哨兵认可。

从剩下的从服务器中挑选出一个作为主服务器，挑选规则如下：

1. 排除已掉线的从服务器
2. 排除五秒内没有回复过INFO命令的从服务器
3. 筛选出与主服务器断线时间最短的从服务器
4. 如果有多个，根据优先级
5. 如果优先级也也一致，那么会根据运行ID排序，选出最小的运行ID

选定完后，就是让其他服务器向这台被选出来的主服务器复制数据。



当原来的主服务器恢复后，哨兵会将它设置为从服务器。



**主从切换**

主从切换是有可能丢数据的，比如主节点刚写入就宕机了，还没来得及同步给从服务器。这种就会出现数据丢失。

哨兵模式只保证高可用，不保证一致性。

某些情况下会出现严重的错误，比如分布式锁。刚刚上锁成功，就发生了主从切换，会出现分布式锁失效的问题。

解决分布式锁的问题，要采用Redlock来解决。

即：向大多数服务器尝试加锁，过半为成功；解锁时要向全部节点解锁。

Redlock的缺陷也很明显，太重了。



**脑裂**

另外一种糟糕的情况是脑裂，即只是主服务器与哨兵的连接断开了，但这时客户端还在向这个主服务器写数据；当主服务器与哨兵的连接恢复后，原来的主服务器被降级成从服务器并丢失了所有数据。



**哨兵脑裂解决方案**

通过两个参数：

`min-slaves-to-write`最少有这个数字的从服务器，才会执行写；否则会拒绝写命令

`min-slaves-max-lag`有`min-slave-to-write`数量的从服务器延迟大于`max-lag`，则拒绝写命令

这样保证，即使与哨兵断开连接，选出来的从服务器的数据和主服务器一致。





