## ThreadLocal 的知识点

#### 为什么要用ThreadLocal

并发场景下，会存在多个线程的同事修改一个共享变量的场景。这就出现了**线程安全的问题**

为了解决线程安全的问题，可以使用加锁的方式，比如使用 **synchronized**或者Lock。但是加锁的方式，可能会导致系统变慢。

还有一种解决方案，就是使用空间换时间的方式，即使用ThreadLocal。使用ThreadLocal类访问共享变量时，会在每个现场的本地，都保存一份共享变量的拷贝副本。线程对共享变量修改时，实际上操作的是这个变量副本，从而保证线程安全。

#### 使用ThreadLocal # set的实现

ThreadLocal的源码

```java
public void set(T value) {,
	Thread t = Thread.currentThread();
	
	// 获取当前线程的ThreadLocalMap
	ThreadLocalMap map = getMap(t);

	if (map != null) {
		// 添加变量
		map.set(this, value);
	} else {
		// 初始化ThreadLocalMap
		createMap(t, value);
	}
}
```

ThreadLocal的源码很简单，但却透漏出不少重要的信息

- 变量存储在ThreadLocalMap中，并且与当前线程有关。
- ThreadLocalMap应该类似于Map的实现。

#### ThreadLocalMap与Thread的关系

**ThreadLocalMap是Thread的成员变量，每个Thread实例对象都拥有自己的ThreadLocalMap**

仅从结构和构造方法中已经能够窥探到ThreadLocalMap的特点：

- ThreadLocalMap底层存储结构是Entry数组；
- 通过ThreadLocal的哈希值取模定位数组下标；
- 构造方法添加变量时，**存储的是原始变量**。

很明显，**ThreadLocalMap是[哈希表](https://link.juejin.cn?target=https%3A%2F%2Fbaike.baidu.com%2Fitem%2F%E5%93%88%E5%B8%8C%E8%A1%A8%2F5981869)的一种实现，ThreadLocal作为Key**，我们可以将ThreadLocalMap看做是“简版”的HashMap。

####  ThreadLocal原理

ThreadLocal，ThreadLocalMap和Thread三者之间的关系：

**ThreadLocal**是作为**ThreadLocalMap**的**key**的，而**ThreadLocalMap**又是**Thread**中的成员变量，**属于每一个Thread的实例对象**。

**创建ThreadLocal对象并存储数据时，会为每个Thread对象创建ThreadLocalMap对象并存储数据，ThreadLocal对象作为key。在每个Thread对象的生命周期内，都可以通过ThreadLocal对象访问到存储的数据。**

## ThreadLocal的内存泄漏

在ThreadLocalMap的源码中可以看到，Entry继承自WeakReference，并且会将ThreadLocal添加到弱引用队列中

我们知道，**弱引用关联的对象只能存活到下一次GC**。如果ThreadLocal没有关联任何强引用，只有Entry上的弱引用的话，发生一次GC后ThreadLocal就会被回收，就会存在ThreadLocalMap上关联Entry，但Entry上没有Key的情况。此时Value依旧关联在ThreadLocalMap上，但无法通过常规手段访问，造成内存泄漏。虽然线程销毁后会释放内存，但在线程执行期间，始终有一块无法访问的内存被占用。