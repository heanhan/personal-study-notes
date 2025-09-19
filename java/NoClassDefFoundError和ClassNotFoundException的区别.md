####  NoClassDefFoundError和ClassNotFoundException的区别

两者的本质区别：

```
ClassNotfoundException 时在编译时 JVM 加载不到类或者找不到类导致。而 NotClassDefFoundError 是在运行时 JVM 加载不到或者找不到类。
```

分析原因：

NoClassDefFoundError 错误发生的原因

为什么会发生 NoClassDefFoundError 错误呢？其实就是和 java 虚拟机的工作原理有关，下面简单介绍 JVM 的类加载机制。

```
类加载三个机制：委托、单一性、可见性。
委托：指加载一个类的请求交给父类型加载器，若父类加载器不可以找到或者加载到，再加载这个类。
单一性：指类加载器不会再次加载父类加载器已经加载过的类。
可见性：子类加载器可以看见父类加载器加载的所有类，而父类加载器不可以看到子类加载器加载的类。
```

JVM 的类加载机制的委托行机制，决定了类加载器只加载一次，子类加载器不会再加载父类型加载器已经加载过得类。