## ReentrantLock锁的解析

ReentrantLock实现了Lock的接口，是一个可以宠辱的独占式的锁，和synchronized关键字类似，不过，ReentrantLock更灵活、更强大，增加了轮询、超时、中断、公平锁等高级功能。

ReentrantLock里面有一个内部类Sync，Sync继承AQS（AbstractQueuedSynchronizer）,添加了锁和释放锁的大部分操作，实际上都是在Sync中实现，Sync有公平锁FairSync和NonfairSync两个子类。

ReentrantLock默认是非公平锁，也可以通过构造器来显示制定使用公平锁。

##### 公平锁和非公平锁有什么区别？

- **公平锁**：锁在被释放之后，先申请的线程先得到锁，性能较差一些，一位公平锁为了保证时间上的绝对顺序，上线问切换更加频繁。
- **非公平锁**：所在释放之后，后申请的线程可能会先获取到锁，是随机或者按照其他优先级排序，性能更好，但是可能会导致某些线程永远无法获取到锁。

##### synchronized 和 ReentrantLock 有什么区别？

- 两者都是可重入锁

  指的是线程可以再次获取自己的内部锁，如果不重入造成锁等待，导致死锁。

- synchronized依赖与JVM而ReentrantLock依赖API