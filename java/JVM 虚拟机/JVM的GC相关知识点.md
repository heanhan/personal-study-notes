### 1、了解java的虚拟机的垃圾回收算法。

jvm的堆空间分为：新生代和老年代、元空间（jdk1.8的叫法）

##### java虚拟机的垃圾回收算法

从年轻代(包括eden、Survivor区域)回收内存被称为 Minor GC，Major GC是清理 永久代的。Full GC 是清理整个堆空间包括年轻代和永久代

