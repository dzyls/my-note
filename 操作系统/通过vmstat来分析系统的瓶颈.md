### 通过vmstat来分析性能瓶颈

---

通过`vmstat`可以查看cpu、memory、disk、swap的一些指标。

```bash
[root@localhost ~]# vmstat
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 3594216   2108 149128    0    0   225    18  118  199  0  1 99  0  0
```

分析 ：

- procs ：进程 。

  - r ：等待执行的任务数
  - b ：IO阻塞的的任务数

- memory ：内存

  - swpd ：正在使用虚拟内存大小【单位k】
  - free
  - buff ：buff是将要写入到磁盘的缓存
  - cache ：cache是从磁盘读取到供cpu读取的缓存

- swap ：内存交换区

  - si ：从交互区写入内存
  - so ：从内存写入交换区

- io :

  - bi ：每秒读入的块数

  - bo ：每秒入的块数

    > 这两个值越大，cpu的wa越大

- system ：

  - in ：每秒中断的次数

  - cs : context switch，每秒上下文切换的次数

    > 这两个值越大，cpu的sy越大

- cpu :

  - us ：用户进程消耗的cpu时间
  - sy ：系统进程消耗的cpu时间
  - id :  空间时间
  - wa ：等待IO时间
  - st ：虚拟机占用CPU时间的百分比

### 如果是cpu瓶颈

---

如果是cpu是瓶颈，就会出现有多个任务在等待cpu执行。

> procs 的 r 数字大【大于CPU核心数】

并且cpu空闲时间也会少

> cpu的id数字小

用户时间很高 ：

> cpu的us数字占比高



### 线程数太多

---

线程数太多或进程数太多，会导致频繁的上下文切换。

表现为 ：

> system的cs切换数字很大。

如果频繁上下文切换，会浪费性能。

可以考虑降低进程数或线程数。



### IO性能瓶颈

---

假设如果出现了IO性能瓶颈 ：

- Procs的b【等待IO的进程数量】数字增大【大于3】
- cpu的wa【等待IO的时间】大【**大于50%**】。【进程可能在进行连续读写的操作，或者IO瓶颈】
- io的bi和bo数字增大【超过2000】



### 内存瓶颈

---

当内存出现不足时，会使用swap区，导致程序缓慢。

因此出现内存瓶颈时，会出现 ：

- memory的swpd不为0
- swap的si和so大于0



### 连锁反应

---

有时还会出现一些连锁反应，比如 ：

- 内存不足，开始使用swap分区，si和so不为0
- 使用swap分区，导致磁盘IO高
- 磁盘IO高，cpu的wa指数高



内存不足有些情况下，会导致性能下降，如果是某些性能要求高的中间件，有可能会造成整个系统雪崩。

比如Redis ：

在没有指定Redis的最大内存占用，Redis内存会一直增大，直到内存不够用，开始使用swap分区，然后Redis性能下降，表现为查询缓存超时。接着就是服务调用超时导致服务雪崩，巨量的访问压力阻塞等待在队列中，进一步占用了机器的缓存，如果监控没有及时监控到，就会~~。



笔者曾经遇到过，服务器使用了内存分区，性能下降导致Redisson的命令队列超长，接着就是OOM了。



### iostat

---

iostat命令则更简单明了 ：

```bash
avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           0.16    0.00    0.17    0.01    0.00   99.67

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda               1.75        51.42        23.93     218109     101506
dm-0              1.28        43.86        23.44     186031      99418
dm-1              0.02         0.58         0.00       2460          0
```





