### ThreadLocal的面试知识点

ThreadLocal是线程本地变量，就是线程的私有变量，不同的线程之间相互隔离，无法共享，相当于每个线程拷贝了一份变量的副本。

目的就是在多线程的环境中，无需加锁，也能保证数据的安全性。

使用demo

```java
public class ThreadLocalDeom{
  //1、创建ThreadLocal
  static ThreadLocal<String> threadLocal =new ThreadLocal<>();
  public static void main(String[] args){
    //2、给ThreadLocal赋值
    threadLocal.set("deom1");
    //3、从ThreadLocal中取值
    String result = threadLocal.get();
    //4、标注：每次使用后需要销毁，防止内存泄露
    threadLocal.remove();
  }
}
```

### ThreadLocal的使用场景

ThreadLocal的应用主要分为两类。

1、避免对象在方法之间的层层传递，打破层次之间的约束，比如用户信息，在很多地方需要用到的，层层传递下去，比较麻烦，这个时候就可以把用户信息放到ThreadLocal中，需要的地方直接使用。

2、拷贝副本，减少初始化的操作，并保证数据安全，比如数据库的连接，spring的事务管理。SimpleDateFormate格式化日期，都是使用ThreadLocal即避免每各线程都初始化使用一个对象，又保证了多线程下的数据安全。

使用ThreadLocal保证SimpleDataFormat格式化日期的线程安全，代码类似下面这样：

```java
public class ThreadLocalDemo {
    // 1. 创建ThreadLocal
    static ThreadLocal<SimpleDateFormat> threadLocal =
            ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));


    public static void main(String[] args){
        IntStream.range(0, 5).forEach(i -> {
            // 创建5个线程，分别从threadLocal取出SimpleDateFormat，然后格式化日期
            new Thread(() -> {
                try {
                    System.out.println(threadLocal.get().parse("2022-11-11 00:00:00"));
                } catch (ParseException e) {
                    throw new RuntimeException(e);
                }
            }).start();
        });
    }

}
```

