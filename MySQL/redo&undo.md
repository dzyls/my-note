### Redo Log 和 Undo Log

MySQL的ACID的实现：

I 隔离性，有锁来实现。

ACD则由redo log和undo log来实现。

**Redo 和 Undo的区别**：

1. redo log是顺序写的，undo log是随机写的
2. undo log 用于实现MVCC，用于事务回滚；redo log用于保证持久性



**Redo Log和binlog的区别**：

1. Redo log是innodb引擎产生的，binlog是由数据库产生的
2. Redo log是记录数据页的物理修改，binlog是记录执行的SQL（statment、row、mix）
3. Redo log在事务执行时不断的并发写入，binlog在事务提交时才写入
4. redo log以512字节存储，和扇区大小一致，写入时可以保证原子性，不需要double write。
5. redo log恢复速度相比binlog快，并且是幂等性的。binlog非幂等性的。



重做日志的格式：

1. redo_log_type ：重做日志的类型
2. space ：表的space id
3. page_no ：page_no 页的偏移量
4. redo log body ：重做日志的



重做日志的LSN:

LSN （Log Seq Number）：

LSN会在redo log中记录，为重做日志的总量，比如当前重做日志的LSN为1000，那么一个事务写入了100字节，LSN就是1100。

LSN也会记录在每个页中，用来判断这个页是否需要恢复操作。比如，数据库重启时，会检测重做日志的LSN，检测是否需要回复操作，如果某页的LSN等于重做日志的LSN那么不需要恢复；如果小于那么需要恢复。

通过`show engine innodb status`可以查询到三个LSN：

`Log Sequence Number`：是当前的LSN

`Log Flush up to`：表示刷到重做日志的LSN

`Last Checkpoint at`：表示刷到磁盘的LSN



在恢复时只需要恢复checkpoint之后的日志即可。

Redo log：记录的是数据页的**物理修改**，可以**并发**写入，所以是**幂等性**的，速度较**快**。



### Undo log

undo log用于事务的回滚，和MVCC的实现。undo log也会产生redo log，用于持久性。

undo log是逻辑日志，并非物理日志，回滚的时候是将数据库**逻辑恢复**到执行前。

对于insert一条，逻辑上是delete一条。

delete一条，则是insert一条。

update，则是将原来的update反过来执行。



### MVCC的实现

MVCC的实现是基于undo log的隐藏字段。

undo log中的行记录有两个隐藏字段可以用于mvcc ：

- trx_id ：当前行的事务ID
- roll_ptr : 回滚指针 ， 通过这一行可以找到上一个版本的记录，版本之间构成版本链



在可重复读的隔离级别下，每个事务都会生成一个 Read View，Read View 中有一个集合，存放的是当前的活跃事务ID，有三个id，Read View 的创建者的事务ID，集合的最小事务ID，下一个事务的ID。



通过行记录的事务ID与Read View做对比，就可以判断当前行是否可见，如果不可见，就沿着回滚指针向下找。

查找规则如下 ：

- 如果当前行的事务ID等于Read View的创建事务ID，则可见
- 如果事务ID小于最小事务ID，则可见
- 如果事务ID大于最大事务ID，不可见
- 如果事务ID大于最小事务ID，小于最大事务ID，则要判断当前行的事务ID是否则活跃的事务ID集合中。如果不在，则说明Read View创建前，此行事务ID已经提交了，则可见。





