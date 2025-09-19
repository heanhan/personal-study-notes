### JVM性能调优的一些观点

#### 一、调优策略

对于GC 的性能指标：吞吐量（工作时间不算gc时间占总时间的比）,和暂停pause(gc发生时应用对外显示无法响应)

#### 1、调优的目的

 调优的最终目的当然增大吞吐量，减少暂停时间咯，映射到GC层面主要关心下面这两点：

  (1)将转移到老年代的对象数量降低到最小。

  (2)减少full GC的执行时间。（尽量减少GC的次数）。

那什么情况对象会转移到老年代，主要有这四种：

(1)新生代对象每经历依次minor gc，年龄会加一，当达到年龄阀值会直接进入老年代。阀值大小一般为15。

(2)Survivor空间中所有对象大小的总和大于survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，而无需等到年龄阀值。

(3)大对象直接进入老年代。

(4)新生代复制算法需要一个survivor区进行轮换备份，如果出现大量对象在minor gc后仍然存活的情况时，就需要老年代进行分配担保，让survivor无法容纳的对象直接进入老年代。









参数设定 





### ParallelGCThreads

其中ParallelGCThreads 参数的默认值是：

- CPU核心数 <= 8，则为 ParallelGCThreads=CPU核心数，比如我的那个旧电脑是4
- CPU核心数 > 8，则为 ParallelGCThreads = CPU核心数 * 5/8 + 3 向下取整
- 16核的情况下，ParallelGCThreads = 13
- 32核的情况下，ParallelGCThreads = 23
- 64核的情况下，ParallelGCThreads = 43
- 72核的情况下，ParallelGCThreads = 48

### ConcGCThreads

ConcGCThreads的默认值则为：

ConcGCThreads = (ParallelGCThreads + 3)/4 向下去整。

- ParallelGCThreads = 1~4时，ConcGCThreads = 1
- ParallelGCThreads = 5~8时，ConcGCThreads = 2
- ParallelGCThreads = 13~16时，ConcGCThreads = 4

## 推荐配置

### 8C16G下的参数配置

综上所述，8C16G下，推荐使用如下的参数设置：

```text
-Xmx12g -Xms12g
-XX:ParallelGCThreads=8
-XX:ConcGCThreads=2
-XX:+UseConcMarkSweepGC
-XX:+CMSClassUnloadingEnabled
-XX:+CMSIncrementalMode
-XX:+CMSScavengeBeforeRemark
-XX:+UseCMSInitiatingOccupancyOnly
-XX:CMSInitiatingOccupancyFraction=70
-XX:CMSFullGCsBeforeCompaction=5
-XX:MaxGCPauseMillis=100  // 按业务情况来定
-XX:+ExplicitGCInvokesConcurrent
-XX:+ExplicitGCInvokesConcurrentAndUnloadsClasses
-XX:+PrintGCTimeStamps
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
```

### 4C8G下的参数配置

如果是4C8G配置下，推荐

```text
-Xmx6g -Xms6g
-XX:ParallelGCThreads=4
-XX:ConcGCThreads=1
// 其他不变。。。。
```

### 2C4G下的参数配置

如果是2C4G配置下，推荐

```text
-Xmx3g -Xms3g
-XX:ParallelGCThreads=2
-XX:ConcGCThreads=1
// 其他不变。。。。
```