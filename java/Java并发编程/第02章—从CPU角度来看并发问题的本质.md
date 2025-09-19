﻿在 Java 语言中，并发一直都是大家所谈论的热门话题，稍微有一点没处理好，就容易导致一些事故。在并发编程领域，考察更多的是程序员们对于**计算机底层原理**的理解，尤其是能够从CPU的角度出发找到问题根源的能力。

为什么这么说？我们一起阅读一段非常简单的代码片段，体验下并发编程的世界中会出现的奇怪现象。

### 经典的 i++ 问题

我们来看看下边这段 Java 代码：

```java

public class ThreadDemo {

    private static int i = 0;


    static class IncrTask implements Runnable {
        CountDownLatch start;
        CountDownLatch end;

        public IncrTask(CountDownLatch start, CountDownLatch end) {
            this.start = start;
            this.end = end;
        }

        @Override
        public void run() {
            try {
                start.await();
                for (int j = 0; j < 100; j++) {
                    i++;
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                end.countDown();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch start = new CountDownLatch(1);
        CountDownLatch end = new CountDownLatch(100);
        for (int i = 0; i < 100; i++) {
            Thread t = new Thread(new IncrTask(start, end));
            t.start();
        }
        start.countDown();
        end.await();
        System.out.println("result is :" + j);
    }
}
```

代码模拟了 100 个线程同时对 i 变量执行 100 次 i++ 的操作，但是最终的执行结果并非是 10000，而且大概率会是一个小于 10000 的数字。通过这个实验我们可以得出一个结论，那就是：**i++ 语句并不是一条原子性的指令，它存在着线程安全问题。**

在 Java 层面来看，i++ 指令似乎已经无法再细分了，但是如果我们更深入一些，从操作系统层面来看 i++ 的话，你会发现，其实它的下层实现，并不是简单的自增操作而已。

为了更深入了解 i++ 在计算机底层发生了什么变化，我用 C++ 写了个简单的案例，通过 https://gcc.godbolt.org/ 平台，将 C++ 代码直接转换为了汇编指令（采用Java语言写还需要先转换为class字节码再到汇编，所以不如直接用 C++ 更方便）。


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ced3f0f90dfc408eb04960ccc027f179~tplv-k3u1fbpfcp-watermark.image?)

具体的汇编指令如下：

```java
i:
        .zero   4
incr():
        push    rbp
        mov     rbp, rsp
        mov     eax, DWORD PTR i[rip]
        add     eax, 1
        mov     DWORD PTR i[rip], eax
        nop
        pop     rbp
        ret
```
关于这段汇编指令，我们可以发现 i++ 的核心部分由三条指令组成：

```java
        mov     eax, DWORD PTR i[rip]
        add     eax, 1
        mov     DWORD PTR i[rip], eax
```
这些指令的大概意思是：将 i 变量挪到 CPU 的一个寄存器中，然后进行自增，接着将 i 移出寄存器放回原先的内存地址中。

看到这里，可能有一些朋友感到疑惑了：**为什么执行这些指令时** **，** **可能会有线程安全问题？**

要理解这个问题，我们得先知道一些计算机的基础知识，比如，什么是寄存器，寄存器和CPU之间有什么联系，等等。所以接下来我们需要先来回顾一下这些基础知识。

### 程序和CPU之间的协作关系

我们都知道，计算机的运作通常都离不开CPU的扶持，一旦离开了CPU，程序就无法运行（*ps：下边主要围绕着以intel旗下x86架构的CPU*）。

如果你有拆过自己的电脑，或者在网上查阅过CPU资料，应该会见过 CPU，就是那个长满了各种引线脚的小方块，下边我将其进行了些许抽象，绘制如下图所示：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5b9c5b729adc41a99a21a95444188505~tplv-k3u1fbpfcp-watermark.image?)

当然，这张图只是将 CPU 的外表模型进行简化，如果将其深入拆解开来，其内部还存在好几个重要的组成部分，下边我将它内部的构造进行核心拆解，主要有以下几个模块，如下图所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3fafa6117cd644cc8069dad226ff6f38~tplv-k3u1fbpfcp-watermark.image?)

#### CPU的几大模块

-   **寄存器**

寄存器是CPU中最需要软件工程师们关心的部分了，因为寄存器中主要存储了从内存中加载的数据（寄存器的运行速度比内存要快好多个级别，准确说是从内存中将数据加载到L1，L2，L3缓存，再到寄存器中），而我们所编写程序在计算机底层转换为汇编语言之后，主要的操控对象其实就是寄存器。例如常见的mov和add指令：


```java
mov eax, DWORD PTR i[rip]
add eax, 1
mov DWORD PTR i[rip], eax
```

其实“寄存器”一词还只是一个大的类目，它的下边还包含了许多种小的子类别，

**我按照存储的数据将它们划分为了两大类** **：** **存储内存地址类寄存器，存储非内存地址寄存器。**

通常单个 CPU 内部就存有成百个寄存器，面对这么多的寄存器如果要逐个了解的话就会让人感到棘手，所以我将它们按照各自的职责划分为了两大类别从而方便大家理解，它们主要有两大类别：

-   **存储内存地址类寄存器**

程序计数器，基址寄存器，变址寄存器。

-   **存储非内存地址寄存器**

累加寄存器，通用寄存器，标志寄存器。

下边我用了一张图来展示它们，如下所示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edc5bd9fbf8143bd884b7c54fd9fa4d8~tplv-k3u1fbpfcp-zoom-1.image)

咦，这六种寄存器的职责分别是什么？别急，后边我会通过一段Java代码案例和绘图的方式，将它们的作用串通起来，方便大家理解。

-   **控制器**

控制器主要是起到一个辅助的功能，它可以帮助CPU做一些指令读取，结果写回等功能，同时它也能根据汇编指令的结果去操控一些计算机的硬件设备。

-   **运算器**

运算器是CPU内部最核心的用于做计算的模块，我们编写的程序在经过多步编译之后最终传达给到CPU的会是一段0和1组成的指令代码，这些指令在控制器的帮助下会将需要计算的数据放入到寄存器中，让运算器去计算。

-   **时钟**

主要是用于记录每次CPU计算的耗时，它的运算单位为ghz，1ghz = 10亿次/秒，通常ghz越高，表示CPU的运算效率越高。

#### 寄存器和程序之间的关系

在了解了CPU的主要组成之后，我们还需要了解它们之间是如何协调运行程序，这样才能更加透彻地去理解并发编程中存在哪些问题需要工程师们注意。下边我们通过一个例子看一下。
这里有一段简单的Java程序：

```java
public class CountDemo {

    public static void compareTest(int a, int b, int[] arr) {
        int t1 = countSum(a);
        int t2 = countSum(b);
        if (t1 > t2) {
            arr[0] = 1;
        } else {
            arr[1] = 1;
        }
    }

    //1～a的求和计算
    public static int countSum(int a) {
        int sum = 0;
        for (int i = 1; i <= a; i++) {
            sum += i;
        }
        return a;
    }


    public static void main(String[] args) {
        int a[] = {-1,-1};
        compareTest(1, 2, a); // ----- code_1
    }
    
}
```
这份程序在IDE工具开发完毕后，实际上会被保存在磁盘当中。

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb01f730fc7946a8a74d9036b01b85a2~tplv-k3u1fbpfcp-watermark.image?)
接着如果要运行程序的话，可以输入一段 Java 指令去运行它，如：javac 和 java 指令。接下来磁盘中的程序会被读取到内存当中，并且进行相关的编译。期间会涉及到多次编译，会先在 jvm 层变成 class 字节码，然后再转换为汇编指令，最后才是到机器码（这也正是将Java代码转换为汇编指令才能认清它背后原理的原因了）。

这些机器码存放的**地址**会被放到一种叫做**程序计数器**的寄存器中，之后**控制器**会到根据程序计数器的地址去读取相关的机器指令，并且将指令读取给**运算器**进行计算。当运行结束之后，**程序计数器的地址就会刷新，让控制器去加载新内存地址的指令给到运算器。**

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a06c2d594900429b80153638b291dcab~tplv-k3u1fbpfcp-watermark.image?)

*ps：有些资料上喜欢将程序计数器和寄存器分成两个部分来说明，但实质上程序计数器本身就是寄存器的一类，因此我个人感觉没有必要将其分开。*

在CountDemo这段代码中虽然涉及到了一个**求和计算，比较计算，函数调用**三个功能，但是在实际运作过程中却牵涉到了文章前边所提到的六种寄存器。

先来看**程序计数器**在代码执行过程中的变化，这里我将各个代码块执行过程中的细节点用图绘制了出来，图中的code_2部分是重复调用countSum函数，所以没有绘制箭头走向，读者们可以根据code_1的指令调用去推理 ：


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c212cfa3cae4237849c6b5a3f4875cd~tplv-k3u1fbpfcp-watermark.image?)


首先 main 函数执行，程序计数器的地址会更新为 main 函数的入口位置，让控制器去加载其指令地址开始执行。接着在准备调用 compareTest 函数的时候，会有一条 **call指令**，将当前的程序计数器地址变更为子函数的入口地址，同理，在 comparetTest 函数内部调用 countSum 函数也是会发送 **call指令**。当子函数执行结束后，便会执行一条 **return指令**，返回到原先执行代码位置的下一条指令位置（call指令和return指令在函数调用的过程中是经常会使用到的）。

我们深入分析 countSum 内部，会发现它包含了求和累积的计算，这里边涉及到了累加寄存器和标志寄存器的使用。**累加寄存器这个很好理解，就是将sum的数值在累加寄存器中不断更新**。而**标志寄存器**其实是用于了判断是否满足跳出循环的逻辑。

我们在 for 循环代码中所编写的 i<=a; 这个逻辑，而计算机底层会通过做差的方式来判断是否满足该条件，也就是变成了：a - i >= 0; 的判断（*这里有些数学中的不等式基础运算的味道～*）。而通过 a-i 计算出来的结果会被记录在**标志寄存器**中的某些个位上，例如下图所示：


![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d1b13d50562c4bd095461206b3c3eb32~tplv-k3u1fbpfcp-watermark.image?)

我们可以将标志寄存器理解为是一个巨大的 bit 数组，不同位置上的 bit 值表示不同的含义，当需要将计算结果记录为负数的话，只需要将 bit[0] 更新为 1 即可。

接下来我们来看看**变址寄存器**和**基址寄存器**，这两个寄存器主要是在数组进行定位元素的过程中会有所使用。为了方便理解，我将这部分用一张图来带大家认识：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/237611d0622449e788c83547af15643a~tplv-k3u1fbpfcp-watermark.image?)

CPU 在对数组这类数据结构的内部元素进行定位的时候会通过基址寄存器的位置 + 变址寄存器的数值进行查询，变址寄存器就有点类似于是数组的索引下标，通过一个相对偏差的数值去对具体位置的定位。

**通用寄存器**这块其实比较好理解 **，** 大家可以将它理解为用于专门存储一些临时变量的公共部分，例如一些临时定义的数字值，对它进行深入了解的意义并不是很大，大家知道有这么一个东西就可以了。


### **再看i++问题**

好了，现在我们终于把计算机底层是如何运作程序的流程给搞清楚了，现在让我们回过头来重新从 CPU的角度去认识下 i++ 指令背后的秘密。

i++ 的核心部分有三条指令组成：

```java
mov eax, DWORD PTR i[rip]
add eax, 1
mov DWORD PTR i[rip], eax
```
mov 指令会先将 i 变量加载到 eax 累加寄存器中，然后在寄存器中做加 1 操作，接着才是将加 1 之后的结果放回到i变量原先的内存地址中，整体流程如下图所示：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb1375214adb4784987946a9677cb86b~tplv-k3u1fbpfcp-watermark.image?)

但是当有多个线程同时使用i++指令的时候就可能会出现如下图的冲突：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a58c94cb5a44498a9468b1c5948786f2~tplv-k3u1fbpfcp-watermark.image?)

不同线程在执行的时候，各自的 eax 累加寄存器中的数值不相同，从而导致最终i被两次更新，但值却不是 2。

而这类现象就是我们常说的线程安全问题了。如果需要解决这类问题，其实只需要保证每个线程在执行i++ 指令的时候都是一个原子操作即可，例如通过加入一道屏障指令，如下图所示：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9923dc4a4392440fb8d7db6b08a06158~tplv-k3u1fbpfcp-watermark.image?)

通过加入一到屏障指令可以保证数据在被多个线程访问的过程中一次只能有一个线程操作它，而且下一个线程会处于等待状态。

通过这段简单的 i++ 指令的运算原理，让我们可以发现，**如今的高级语言已经将一些系统底层的细节步骤进行了封装** **，** 如果对于它背后的机制没有专门研究的话，在一些复杂场景中很可能会产生一些意想不到的情况。所以，我们得能够深入系统底层，理解CPU的知识，从CPU的角度出发来看并发问题。

### 课后小结

好了，在本节中，我们重点梳理了 CPU 的知识点，并且通过一些简单的案例展示，带大家从 CPU 的角度去了解执行程序过程中为什么可能会产生并发问题。

最后是本章节的知识点归纳，希望大家可以下去深入消化一下：

-   操作内部CPU底层的核心组成；
-   程序是如何在CPU的辅助下去运作的；
-   操作系统内部的核心寄存器种类，及它们的职责；
-   并发访问数据信息时，操作系统底层可能会发生哪些问题。

### 课后思考

如果让你来设计一段程序让 CPU 从内存中读取数据到寄存器，你会考虑到哪些问题点？欢迎大家在底下评论区进行留言讨论，我会在下一章节中给出自己的思考。




