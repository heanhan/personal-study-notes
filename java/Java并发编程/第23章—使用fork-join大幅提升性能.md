从 JDK7 版本开始，ForkJoin 线程池开始对各个开发者开放，这款线程池的特别之处在于，它不单单执行任务，还会将一个大的任务拆解为多个小任务再执行。举个例子，例如我们需要计算 1-10000 的求和，使用传统的单线程去计算的话，就是顺序执行：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8af800aab98d40719ef6a744790560cd~tplv-k3u1fbpfcp-watermark.image?)

然而，如果我们尝试将这个求和任务分解成好几个阶段，例如将计算的任务分解为计算其中的 0-1250，1251-2500，2501-3750 ... (后边依次类推) ，同时定义多个线程，分别负责不同步骤的计算任务，最终再将计算结果统一汇总，这种思路的实现需要我们定义多个线程去实现，例如下图：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/46e35f88b6994ac9bd96614cdd6eebd0~tplv-k3u1fbpfcp-watermark.image?)

在 ForkJoin 线程池组件中，则充分体现了先拆解后合并的设计思想，下边让我们一起来认识下这个组件的特性。

## **如何理解ForkJoinPool**

其实 ForkJoinPool 的原理非常好理解，我们可以根据名字来认识它的作用。

**Fork** **，** 顾名思义，在英语中 Fork 一词有分开的意思，我们可以理解为将一个任务拆解开来的意思。

**Join** **，** 从英文翻译过来是合并的意思，可以理解为将一开始拆解的任务的计算结果统一合并起来。

ForkJoinPool 的作用既包含了拆解任务，也包含了合并结果的步骤，它之所以能达成这样的效果，主要和它内部的多任务队列有关。下边我绘制了一张 ForkJoinPool 底层的原理图：


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/639c68d9400a4059afc60826a23ba9ff~tplv-k3u1fbpfcp-watermark.image?)

在 ForkjoinPool 的内部，每个请求的任务都会通过决策被放入到某条任务队列中，接着会有一批提前准备好的线程从各个队列中去获取任务，所以 ForkJoinPool 利用了多线程+多队列的方式来提升任务的处理速率。

**那么每个线程都只会负责一条队列中的任务吗？**

关于这一点，答案是否定的，当 ForkJoinPool 中的线程处于空闲的时候，它们会自觉去扫描其他的任务队列，查看是否有可以执行的任务，如果发现有，则会 **“窃取”** 其他队列上的任务。这个机制也被称作为 ForkJoinPool 的 **窃取机制。**

ForkJoinPool 底层的任务队列在弹出任务时，其实也有些讲究，为了减少线程抢夺带来的性能损耗，正常任务的获取会从队头弹出，通过“窃取”方式获取的任务会从队尾弹出。

采用了这种窃取机制之后，其实有好有坏，我整理了下它的优缺点，具体如下所示。

**工作窃取算法优点** **：**

-   充分利用线程进行并行计算；
-   减少了线程间的竞争。

**工作窃取算法缺点**

-   在某些情况下会存在竞争（双端队列中只有一个任务）；
-   消耗了更多的系统资源。

现在我们对 ForkJoinPool 有了一个大概的认识，接下来来看看如何在实际应用中使用 ForkJoinPool。

## **数字求和案例**

下边是一个基于 Fork-Join 技术去实现的数组求和功能，对 10000 个随机数字进行累加操作。

```java
public class ForkJoinSumDemo {

    public static AtomicInteger countTime = new AtomicInteger(0);

    static class ArraySumTask extends RecursiveTask<Long> {

        private static final int grain = 2000;
        private int begin, end;
        private long[] src;

        public ArraySumTask(int begin, int end, long[] src) {
            this.begin = begin;
            this.end = end;
            this.src = src;
        }

        @Override
        protected Long compute() {
            Long count = 0L;
            if (end - begin <= grain) {
                for (int i = begin; i < end; i++) {
                    count += src[i];
                }
            } else {
                int mid = (begin + end) / 2;
                //将大任务分解为多个小任务去执行
                RecursiveTask left = new ArraySumTask(begin, mid, src);
                RecursiveTask right = new ArraySumTask(mid, end, src);
                invokeAll(left, right);
                long leftJoin = (long) left.join();
                long rightJoin = (long) right.join();
                count = leftJoin + rightJoin;
            }
            System.out.println("计算第" + countTime.getAndIncrement() + "次");
            return count;
        }
    }

    public static Random random = new Random();

    public static void main(String[] args) {
        int total = 10000;
        long[] data = new long[total];
        long testCount = 0;
        //随机填充数据
        for (int i = 0; i < total; i++) {
            data[i] = random.nextInt(5);
            testCount += data[i];
        }
        ArraySumTask arraySumTask = new ArraySumTask(0, data.length, data);
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        long sum = forkJoinPool.invoke(arraySumTask);
        System.out.println(sum);
        System.out.println(testCount);
    }
}
```

在这个案例中，我们将对数字的累加操作封装在另一个叫做 **ArraySumTask**的任务当中，当 **ForkJoinPool** 第一次调用 invoke 函数去触发任务时，就会触发到 ArraySumTask 的 compute 方法。从上边的代码来看，我们可以发现，每次调用 compute 方法都会对当前数组的长度做一次判断，如果任务持有的数组个数大于 2000 ，则会分解为更小的任务。下边我绘制了一张图去解释这个流程的底层思路：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/45d340aeafa6484c9919be6f787c8da2~tplv-k3u1fbpfcp-watermark.image?)

首先是 ForkJoinPool 调用 invoke 函数，去触发任务的 compute 方法，但是在这个 compute 方法中会有一层递归的逻辑，在递归中不断地拆解这 1w 个数字，当被拆解达到预期时候，才会进入到具体的逻辑中。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/01af961c038f4e55be4def86815fbbaa~tplv-k3u1fbpfcp-watermark.image?)
## **数组中求最大值案例**

其实这类案例场景在实际应用中也是经常会应用到的。

```java
package 并发编程18.forkjoin使用案例;

import java.util.Arrays;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

/**
* 使用forkjoin进行任务计算
* 查找一批数据中最大的数字
*
*  @Author  linhao
*  @Date  created in 8:02 上午 2022/9/1
*/
public class ForkJoinFindBigNumberDemo {

    public static class MyTask extends RecursiveTask<Integer> {
        int[] innerSrc;

        public MyTask(int[] innerSrc) {
            this.innerSrc = innerSrc;
        }


        @Override
        protected Integer compute() {
            if (innerSrc.length < 2) {
                if (innerSrc.length == 1) {
                    return innerSrc[0];
                } else {
                    return innerSrc[0] > innerSrc[1] ? innerSrc[0] : innerSrc[1];
                }
            } else {
                int mid = innerSrc.length / 2;
                MyTask leftTask = new MyTask(Arrays.copyOf(innerSrc, mid));
                MyTask rightTask = new MyTask(Arrays.copyOfRange(innerSrc, mid, innerSrc.length));
                invokeAll(leftTask, rightTask);
                return leftTask.join() > rightTask.join() ? leftTask.join() : rightTask.join();
            }
        }
    }


    public static void main(String[] args) {
        int[] src = {1, 2, 3, 4, 5, 6, 7, 18, 9, 10, 11, 12};
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        MyTask totalTask = new MyTask(src);
        forkJoinPool.invoke(totalTask);
        System.out.println("the max number is " + totalTask.join());
    }
}
```

## **使用ForkJoinPool时需要注意什么？**

**submit 与 invoke 提交的区别** **？**

-   invoke 是同步执行，调用之后需要等待任务完成，才能执行后面的代码。
-   submit 是异步执行，只有在 Future 调用 get 时会阻塞。

**继承 RecursiveTask 与 RecursiveAction的区别？**

-   继承 RecursiveTask：适用于有返回值的场景。
-   继承 RecursiveAction：适合于没有返回值的场景。

**子任务调用 fork 与 invokeAll 的区别？**

-   fork：让子线程自己去完成任务，父线程监督子线程执行，浪费父线程。
-   invokeAll：子父线程共同完成任务，可以更好地利用线程池。

## **工作中如何使用 ForkJoin？**

其实在工作中，我们一般不会自己手动构建使用 ForkJoinPool，相信大多数使用到 ForkJoinPool 的时候，是在使用 Jdk8 的 parallelStream 时。例如下边这段代码：

```java
void batchInsert(List<Long> userIdList) {
    userIdList.stream().forEach(userId->{
        //需要先写入A表，然后获取写入的数据主键id
        int id = insertTA();
        //将主键id和userId写入到B表中
        boolean result = insertTB(userId,id);
        if(result) {
            //todo 业务逻辑
        }
    });
}
```

这段代码中采用的是 stream 这种串行流来执行写入操作，假设每次执行插入操作需要耗时 4ms，那么当有 100 个用户 id 的时候，整个函数的执行时长就需要等 400ms。这样的耗时对响应时间比较敏感的场景有一定影响，所以我们不妨尝试将 stream 流换成 parallelStream 流，例如下边所示：

```java
void batchInsert(List<Long> idList) {
    idList.parallelStream().forEach(x->{
        int id = insertTA();
        boolean result = insertTB(x,id);
        if(result) {
            //todo 业务逻辑
}
    });
}
```

使用 parallelStream 流之所以能提升效率，在于其底层使用了 ForkJoin 线程池，将插入任务分解成了多个，然后散落在不同的队列中并且交给多个线程使用，从而提升计算的速率。

但是使用了 parallelStream 之后，我们需要注意以下几个点：

-   **是否存在线程安全问题** **？**

因为原先要执行的任务被拆解为了多个小任务，并且交给多个线程去处理，所以如果有共享数据访问操作的话，这块就有可能会存在线程安全问题，需要注意。

-   **parallelStream所使用的线程池核心数是否合理？**

parallelStream 默认使用的线程池核心数默认为: 当前运行环境的 CPU 核数 - 1，但是如果我们部署的服务器是单核机器，那么使用 parallelStream 的意义并不大，额外的线程切换反而可能会有更多的性能开销。


## **课后小结**

本章节中我们重点介绍了 ForkJoin 技术的一些使用案例，通常会用于一些大型的离线计算任务中，关于 ForkJoin 我们可以把它当成一个单机版本的 MapReduce 框架，将一个巨大的任务拆解为许多个小任务进行计算，最终再将每个任务的结果进行汇总。

最后，在使用 ForkJoin 的时候，需要注意它的一些内部 api 特性，例如 invoke、submit、invokeAll等。

## **课后思考**

**上节课思考**

上节课的结尾，我们留下了一道思考题。关于常见的缓存算法 FIFO、LRU、LFU，W-TinyLFU 的介绍，我整理了一下，大致如下：

**FIFO 算法**

这类算法通常会被应用于缓存队列中，当一个查询请求命中了某个元素之后，便会将它放入到队列中，后续的命中元素也是以此类推，直到队列满了之后，老的元素就会被弹出清除。具体流程如下图所示：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/860125ed3ca4417bb656a2e2028f659a~tplv-k3u1fbpfcp-watermark.image?)

**不足点：老元素如果某段时间没有访问，就会被放置到队列尾部，即使重新访问也依然在队列尾部，当元素面临淘汰的时候，老元素（即使是热点元素）会被误删。**

**LRU（Least Recently Used ）算法**

当缓存队列内部已经存在了一批元素之后，后期请求如果命中了队列中的某个元素，那么这个元素就会被前置到队列的头部，从而降低后期被清空的风险。


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f212ac73c224c809f94f567cbeaa342~tplv-k3u1fbpfcp-watermark.image?)

**不足点：LRU 算法存在着“缓存污染”的情况，需要避免，当突然有一批非热点元素查询打入，大量的非热点数据就会被加载到缓存队列中，从而把真正的热点元素给“挤出去”。**


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/721b6b89d10a454d9e23736726765515~tplv-k3u1fbpfcp-watermark.image?)

所以为了避免这类缓存污染的问题，后来又出现了一种 LFU 的策略。

**LFU（Least Frequently Used）算法**

LFU 策略就是会在实现缓存队列的基础上额外新增一个内存空间用于记录缓存中每个元素的访问次数，然后根据访问频率来判定哪些元素可以保留，哪些元素需要被删除。

这类算法存在一个很大的弊端，就是需要耗费额外的空间来存储每个元素的访问频率，因此随着缓存元素的数目不断增大，计数器的个数也在不断地增大。


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed2fc42356e14a1b960e49cb8f2caa9b~tplv-k3u1fbpfcp-watermark.image?)

**不足点**

使用 LFU 算法也会存在某些程度上的“缓存污染”影响，例如当某天搞秒杀活动，突然一批数据被访问了上千万次，但是第二天这批数据就不再访问了，但是又由于之前秒杀活动导致这批数据的访问基数太过高，导致一直无法清空，所以会一直占用着本地缓存的空间。

**W-TinyLFU（Window Tiny LFU）算法**

W-TinyLFU 实际是一种缓存置换算法，直译过来大概是带一个窗口的 tiny 版本的 LFU 实现； 主要用三段 LRU，一个布隆过滤器、一个`cm-sketch`计数器，兼顾了 LFU 和 LRU 的优点。这种缓存设计算法被如今大名鼎鼎的 Caffeine 所使用。

W-TinyLFU 的基础底层其实是使用了 TinyLFU，在 TinyLFU 中，它主要通过使用位图的方式来压缩每个 key 的访问频率，将 key 取 hash 值获得它的位下标，然后用这个下标来找频率。


![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/172ff368da82450389c78624fac3e615~tplv-k3u1fbpfcp-watermark.image?)

另外在内存存储上，W-TinyLFU 使用的是分区存储，这一点有些类似于 MySQL 中 buffer pool的lru 链表，5/8 存 young page，3/8 存 old page。

关于 W-TinyLFU 的更多细节，大家可以深入研究下它的学术论文，地址如下：

```
https://arxiv.org/pdf/1512.00727.pdf
```

**本节课思考**

前边的章节中，有小伙伴提问到使用线程池这种方式处理异步请求虽然可以提升效率，但是容易丢失，大家有什么好的解决方案吗？欢迎在评论区中给出解答。

