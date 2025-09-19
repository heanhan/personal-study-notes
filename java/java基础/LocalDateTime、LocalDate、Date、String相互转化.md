### LocalDateTime、LocalDate、Date、String相互转化

#### 1、LocalDateTime、LocalDate、Date三者联系

这里先说一下，为什么日期有`Date`了，还在`JDK8`中推出了`LocalDateTime、LocalDate`呢？

原因Date：

1. 非线程安全的方法

   Date类的大部分方法都不是线程安全的，比如`setYear()、setMonth()、setDate()、setHours()、setMinutes()、setSeconds()`等方法。这些方法都可以修改Date对象的内部状态。如果`多个线程`同时访问和修改`同一个Date`对象，就会发生竞态条件，导致程序出现错误的结果。

2. 全局变量的使用

   Date类有两个静态变量，分别是`DateParser`和`CalendarSystem`。这两个变量是`全局共享`的，如果多个线程同时访问和修改这两个变量，也会导致程序出现竞态条件。

因此，如果需要在线程中使用日期时间相关的操作，建议使用线程安全的日期时间类，比如JDK8中引入的新日期时间API中的`LocalDateTime、LocalDate`等类，或者使用线程安全的`DateFormat和Calendar`类。

我们在说一下LocalDateTime他们是怎么实现线程安全的：

LocalDateTime它是由LocalDate和LocalTime两个不可变的类组成的。LocalDate和LocalTime各自都是线程安全的，它们的时间信息都是基于`UTC`（协调世界时）计算的，并且不依赖于系统的时区设置。LocalDateTime也是一样，它是由系统时区和UTC计算得到的。

这些类主要是使用了以下两个技术来解决线程安全问题：

1. 不可变性：这些类都是不可变的，一旦创建后，就不能再被修改。因此，就不存在并发修改的问题了。
2. 线程封闭性：这些类的构造方法都是线程安全的，并且不允许外部修改其中的状态。因此，就不需要通过锁或其他机制来保护它们的状态。

综上所述，Java 8中的新日期时间API通过`不可变性`和`线程封闭性`等技术，有效地解决了线程安全问题。这使得开发者们可以更加安全和便利地在多线程环境下使用日期时间类。

#### 相互转换例子

#####  LocalDate转String

```java
String dateStr = LocalDate.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));
System.out.println("当前字符串日期：" + dateStr);
```

#####  String转LocalDate

```java
LocalDate date = LocalDate.parse(dateStr);
System.out.println("日期对象：" + date); 
```

##### LocalDateTime转String

```java
String dateTimeStr = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
System.out.println("当前字符串日期时间：" + dateTimeStr);
```

##### String转LocalDateTime

```java
LocalDateTime dateTime = LocalDateTime.parse(dateTimeStr, DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
System.out.println("当前日期时间对象：" + dateTime);
```

##### LocalDate转Date

```java
Date dateNew1 = Date.from(date.atStartOfDay().atZone(ZoneId.systemDefault()).toInstant());
System.out.println("当前日期对象转date：" + dateNew1);
```

##### LocalDateTime转Date

```java
Date dateNew2 = Date.from(dateTime.atZone(ZoneId.systemDefault()).toInstant());
System.out.println("当前日期时间对象转date：" + dateNew2);
```

##### Date转LocalDate

```java
LocalDate localDate = dateNew2.toInstant().atZone(ZoneId.systemDefault()).toLocalDate();
System.out.println("当前date转日期对象：" + localDate);
```

##### Date转LocalDateTime

```java
LocalDateTime localDateTime = dateNew2.toInstant().atZone(ZoneId.systemDefault()).toLocalDateTime();
System.out.println("当前date转日期时间对象：" + localDateTime);
```

##### Date相互转String

```java
DateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
System.out.println("date转String字符串:" + df.format(dateNew2));

DateFormat df1 = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
System.out.println("String字符串转date:" + df1.parse(dateTimeStr));
```

**需要注意的是**

`SimpleDateFormat`是线程不安全的类，不适用于多线程环境，所以在实际开发中需要注意线程安全问题。可以考虑使用`ThreadLocal`来解决线程安全问题