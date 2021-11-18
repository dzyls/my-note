## ReadWriteLock的缺点

`ReadWriteLock`解决了并发读的问题，但还是有一点缺点，就是当加了读锁时，写锁是会阻塞等待的。也就是读锁与写锁是互斥的，是悲观锁。

**如果是读多写少的情况下，使用悲观读，效率就很低**

在保证线程安全的前提下，要进一步提高并发，可以提供乐观读，即读时不加锁，使用乐观读不与写锁互斥。读完之后，使用时再校验是否加过写锁，如果要保证一致性，那就再重试或者升级为悲观读。

JDK8就提供了这样一个乐观读的工具类 ：`StampedLock`



## StampedLock

StampedLock是一个**不可重入**锁。

加锁：

```java
long writeStamp = stampedLock.writeLock();
long readStamp = stampedLock.readLock();
// 乐观读
long optimisticRead = stampedLock.tryOptimisticRead();
```

读锁写锁是互斥的，但乐观读不是互斥的，如果想要确保数据准确，可以使用`validate`来校验是否存在写锁，如果存在写锁，则使用悲观读。



解锁：

```java
stampedLock.unlockWrite(writeStamp);
stampedLock.unlockRead(readStamp);
```



验证：

```java
boolean validate = stampedLock.validate(stamped)
```

这个是校验从邮戳到当前是否加过写锁，如果加过写锁，则要重试或使用悲观锁。



转换：

```java
long writeStamp = stampedLock.tryConvertToWriteLock(stamped);
long readStamp = stampedLock.tryConvertToReadLock(writeStamp);
```



从以上来看，StampedLock是比可重入锁复杂很多，但是性能是比重入锁高很多。

缺点如：

- 不可重入，如果一个线程中调用两次加锁，那么会自己把自己锁住了
- 编码复杂
- 不支持wait/notify机制



## 原理

`StampedLock`底层的原理是使用了一个Long类型的state，将这个long值的第`0-7`位记录当前读线程的数量，第`8-63`位记录写锁的加锁次数。

- 为什么要记录写锁的加锁次数？

记录写锁的加锁次数，是为了乐观读时校验是否有线程加了写锁修改过值。

- 为什么要记录读线程的数量？

方便加写锁的时候判断当前是否有线程在悲观读。



## 底层实现

stampedLock底层有个链表，用于存储等待锁的线程。

```java
static final class WNode {
    volatile WNode prev;
    volatile WNode next;
    volatile WNode cowait;    // list of linked readers
    volatile Thread thread;   // non-null while possibly parked
    volatile int status;      // 0, WAITING, or CANCELLED
    final int mode;           // RMODE or WMODE
    WNode(int m, WNode p) { mode = m; prev = p; }
}
```



除此之外，还有一个long类型的`state`比较重要。



加锁流程 ：

```java
private static final int LG_READERS = 7;
private static final long RUNIT = 1L;
private static final long WBIT  = 1L << LG_READERS;

public long writeLock() {
    long s, next;  // bypass acquireWrite in fully unlocked case only
    return ((((s = state) & ABITS) == 0L &&
             U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
            next : acquireWrite(false, 0L));
}
```

- state & ABITS是判断当前是否存在读锁或者写锁，如果不存在，才能进入下一步
- 对state进行CAS替换，替换的值为s + WBIT。
- 如果成功，则返回值；如果不成功，则入队。



可以看到，每次加写锁都是`state += 1L<<7`，每次相加之后，第7位之后的值，就是加写锁的次数。即第`8-63`位是添加写锁的次数，每次乐观读时，只需要判断获取stamp和校验stamp之间有没有加写锁。



校验 ：

```java
private static final long RBITS = WBIT - 1L;
private static final long SBITS = ~RBITS; // note overlap with ABITS

public boolean validate(long stamp) {
    U.loadFence();
    return (stamp & SBITS) == (state & SBITS);
}
```

- 从上面得知，WBIT 为`10000000`;那么`RBIT=WBIT - 1`就等于`011111111`；SBITS就等于RBIT取反
- `stamp & SBITS`就是获取第七位到第六十三位之后的值，与`state&SBITS`相比就是判断stamp之后有没有加过写锁



**accquireWrite**

`accquireWrite`的逻辑看起来很长，但步骤就几个：

1. 如果当前没有读锁也没有写锁，那么就CAS尝试加锁
2. 自旋CAS尝试入队，如果队列没有初始化，则初始化
3. 如果当前节点的前置节点是头节点，说明快到自己了，自旋抢锁；
4. 如果当前节点的前置节点不是头结点，则唤醒头结点的读线程
5. 没抢到就park等待



**unlockWrite**

```java
public void unlockWrite(long stamp) {
    WNode h;
    // 检查状态
    if (state != stamp || (stamp & WBIT) == 0L)
        throw new IllegalMonitorStateException();
    // 判断写的标志位是否已经溢出了，如果溢出了，则重置
    state = (stamp += WBIT) == 0L ? ORIGIN : stamp;
    if ((h = whead) != null && h.status != 0)
        // 唤醒下一个线程
        release(h);
}
```



## 注意事项

**乐观读一定要校验**

使用乐观读时，要注意按照模板来 ：

```java
public void optimisticRead(){
    // 使用乐观读
    long stamp = lock.tryOptimisticRead();
	// 此处还要判断stamp是否为0L;亦或者不判断，等到校验时直接升级为悲观读
    if(stamp == 0){
        // return or retry
    }
	// 拷贝共享资源到本地变量
    copyVar2ThreadMemory();
    if(!lock.validate(stamp)){
        // 如果校验不通过，则升级为悲观读
        stamp = lock.readLock();
        try{
            copyVar2ThreadMemory();
        }finally{
            // 解锁一定一定要放在finally
            lock.unlockRead(stamp);
        }
    }
    // 使用本地变量做点啥
    useThreadMemoryVar();
}
```

可以看到，使用悲观读，编码要比使用读写锁复杂太多了。



**ReadLock和WriteLock不可中断**

ReadLock和WriteLock使用了大量的CAS自旋操作，如果使用不当，会导致CPU占用彪高。



原理解析：

ReadLock和WriteLock内部使用了park()函数等待，如果使用中断，park函数会立即返回，不会抛出异常，然后又进入自旋操作了，当自旋没有抢到锁时，会再次调用park()函数等待，但是由于调用过中断，中断的标记还在，因此park()会立即返回，再次自旋，因此当调用中断后，park（）的就不会等待了，会一直自旋。



因此，**切记不要对StampedLock的readLock和writeLock进行中断操作**。

如果一定要使用可中断的读写锁，可以使用`readLockInterruptibly`和`writeLockInterruptibly`。这两个函数会对线程是否被中断过进行判断，当发现中断过就不会自旋会抛出异常。



**不可重入**

StampedLock是不可重入锁，没有使用AQS。

千万千万不要自己锁住自己了。



**不支持Condition和wait/notify**



**适合读多写少的场景**

如果写多读少的场景，不要使用StampedLock。



## long类型的妙用

StampedLock将写锁的次数和读线程的个数，记录在一个long类型变量中，这样可以很方便的进行CAS操作。

Java工具类的BitSet底层也是用Long类型的数组来做位数组，在代码实现中，使用long转换为二进制再进行处理会节省很多空间。