# JVM 常用参数

## 开启TLAB

开启开启TLAB后,为没个线程预先分配一块内存，创建对象优先使用TLAB内存分配空间。

```shell
XX:&#43;UseTLAB  # 开启TLAB，默认开启
XX:TLABSize  # 设置缓存区大小
```

## 开启指针压缩

在64位平台的HotSpot中使用32位指针，内存使用会多出1.5倍左右，使用较大指针在主内存和缓存之间移动数据， 占用较大宽带，同时GC也会承受较大压力，默认开启！

```shell
XX:&#43;UseCompressedOops
```

## 开启逃逸分析

jdk1.7 后默认开启。

```shell
-XX:&#43;DoEscapeAnalysis
```

## 开启标量替换

jdk1.7 后默认开启。

```shell
-XX:&#43;EliminateAllocations
```

## Serial收集器

```shell
-XX:&#43;UseSerialGC -XX:&#43;UseSerialOldGC
```

## Parallel Scavenge 收集器

```shell
-XX:&#43;UseParallelGC -XX:&#43;UseParallelOldGC
```

## ParNew 收集器

```shell
-XX:&#43;UseParNewGC
```

## CMS收集器

```shell
-XX:&#43;UseConcMarkSweepGC
```

-  -XX:&#43;UseConcMarkSweepGC：启用cms 

-  -XX:ConcGCThreads：并发的GC线程数 

- -XX:&#43;UseCMSCompactAtFullCollection：FullGC之后做压缩整理（减少碎片） 

- -XX:CMSFullGCsBeforeCompaction：多少次FullGC之后压缩一次，默认是0，代表每次FullGC后都会压缩一 次

-  -XX:CMSInitiatingOccupancyFraction: 当老年代使用达到该比例时会触发FullGC（默认是92，这是百分比） 

- -XX:&#43;UseCMSInitiatingOccupancyOnly：只使用设定的回收阈值(-XX:CMSInitiatingOccupancyFraction设 定的值)，如果不指定，JVM仅在第一次使用设定值，后续则会自动调整 

- -XX:&#43;CMSScavengeBeforeRemark：在CMS GC前启动一次minor gc，目的在于减少老年代对年轻代的引 用，降低CMS GC的标记阶段时的开销，一般CMS的GC耗时 80%都在标记阶段 

- -XX:&#43;CMSParallellnitialMarkEnabled：表示在初始标记的时候多线程执行，缩短STW 

- -XX:&#43;CMSParallelRemarkEnabled：在重新标记的时候多线程执行，缩短STW; 

## G1收集器

- -XX:&#43;UseG1GC:使用G1收集器 

- -XX:ParallelGCThreads:指定GC工作的线程数量 

- -XX:G1HeapRegionSize:指定分区大小(1MB~32MB，且必须是2的N次幂)，默认将整堆划分为2048个分区 

- -XX:MaxGCPauseMillis:目标暂停时间(默认200ms) 

- -XX:G1NewSizePercent:新生代内存初始空间(默认整堆5%) 

- -XX:G1MaxNewSizePercent:新生代内存最大空间 

- -XX:TargetSurvivorRatio:Survivor区的填充容量(默认50%)，Survivor区域里的一批对象(年龄1&#43;年龄2&#43;年龄n的多个 年龄对象)总和超过了Survivor区域的50%，此时就会把年龄n(含)以上的对象都放入老年代 

- -XX:MaxTenuringThreshold:最大年龄阈值(默认15) 
-  -XX:InitiatingHeapOccupancyPercent:老年代占用空间达到整堆内存阈值(默认45%)，则执行新生代和老年代的混合 收集(**MixedGC**)，比如我们之前说的堆默认有2048个region，如果有接近1000个region都是老年代的region，则可能 就要触发MixedGC了 

- -XX:G1MixedGCLiveThresholdPercent(默认85%) region中的存活对象低于这个值时才会回收该region，如果超过这 个值，存活对象过多，回收的的意义不大。 

- -XX:G1MixedGCCountTarget:在一次回收过程中指定做几次筛选回收(默认8次)，在最后一个筛选回收阶段可以回收一 会，然后暂停回收，恢复系统运行，一会再开始回收，这样可以让系统不至于单次停顿时间过长。 

- -XX:G1HeapWastePercent(默认5%): gc过程中空出来的region是否充足阈值，在混合回收的时候，对Region回收都 是基于复制算法进行的，都是把要回收的Region里的存活对象放入其他Region，然后这个Region中的垃圾对象全部清 理掉，这样的话在回收过程就会不断空出来新的Region，一旦空闲出来的Region数量达到了堆内存的5%，此时就会立 即停止混合回收，意味着本次混合回收就结束了。 

## 普通应用CMS GC 设置

- 设置内存分配参数, 设置元空间初始大小和最大值相等可加快应用的启动。
- 设置垃圾收集器, 年轻代用ParNew,老年期用CMS。
- 打印GC日志，使用滚动方式打印。

```shell
-Xms2048M -Xmx2048M -XX:MetaspaceSize=256M -XX:MaxMetaspaceSize=256M 
-XX:&#43;UseParNewGC -XX:&#43;UseConcMarkSweepGC -XX:CMSInitiatingOccupancyFraction=92 -XX:&#43;UseCMSInitiatingOccupancyOnly
-XX:&#43;PrintGCDetails -XX:&#43;PrintGCDateStamps -XX:&#43;PrintGCTimeStamps -XX:&#43;PrintGCCause -XX:&#43;UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M -Xloggc:/tmp/gc-cms-%t.log 
```

## Kafka JVM 参数设置

LinkedIn 60个broker使用G1收集器

```shell
-Xmx6g -Xms6g -XX:MetaspaceSize=96m -XX:&#43;UseG1GC
  -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M
  -XX:MinMetaspaceFreeRatio=50 -XX:MaxMetaspaceFreeRatio=80 -XX:&#43;ExplicitGCInvokesConcurrent
```

- 60 brokers
- 50k partitions (replication factor 2)
- 800k messages/sec in
- 300 MB/sec inbound, 1 GB/sec&#43; outbound

## Rocketmq 推荐参数设置

推荐 8G 内存，使用G1收集器

```shell
-Xms8g -Xmx8g -Xmn4g -XX:&#43;UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30
-XX:MaxGCPauseMillis -XX:&#43;UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m -Xloggc:/dev/shm/mq_gc_%p.log
```



---

> 作者: [telzhou618](https://github.com/telzhou618)  
> URL: https://telzhou618.github.io/posts/jvm/  

