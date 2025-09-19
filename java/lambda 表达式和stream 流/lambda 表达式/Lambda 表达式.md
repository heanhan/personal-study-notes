### 1、Lambda 表达式

#### 1.1 lambda 表达式的简介

Lambda 表达式是 jdk8 的一个新特性，可以取代大部分的匿名内部类，写出更优雅的 java 代码，尤其在集合的遍历和其他集合操作中，可以极大的优化代码结构。jdk 也提供的大量的内置函数式的接口，使得 Lambda 表达式的运用更加方便和高效。

#### 1.2 对接口的要求。

虽然使用 lambda 表达式可以对某些接口进行简单的实现，但不是所有的接口都可以使用 lambda 表达式来实现。lambda 规定接口只能有一个需要实现的方法，不是规定接口中只能有一个方法，jdk8 中有另一个新特性：default 被一个 default 修饰的方法会有默认的实现，不是不学实现的方法。所以不影响 lambda 表达式的使用。

#### 1.3 @Functionallnterface

修饰函数式接口的，要求接口中的抽象方法只有一个，这个注解往往和 lambda 表达式一起出现。（起验证作用。）

#### 1.4 Lambda 基础语法

```java
public class LambdaInterface {

    /**
     * 无参无返回值
     */
    @FunctionalInterface
    interface NoReturnNoParam{
        void method();
    }

    /**
     * 有一个参数无返回值
     */
    @FunctionalInterface
    interface NoReturnOneParam{
        void method(int a);
    }

    /**
     * 多参数无返回值
     */
    @FunctionalInterface
    interface NoReturnMultipleParam{
        void method(int a,int b);
    }


    /**
     * 无参数有返回值
     */
    @FunctionalInterface
    interface ReturnNoParam{
        int  method();
    }

    /**
     * 一个参数有返回值
     */
    @FunctionalInterface
    interface ReturnOneParam{
        int  method(int a);
    }

    /**
     * 多个参数有返回值
     */
    @FunctionalInterface
    interface ReturnMultiParam{
        int  method(int a,int b);
    }


    /**
     * 测试lambda表达式的使用
     */
    public static void main(String[] args) {

        /**
         * 没有参数没有返回值。
         */
        NoReturnNoParam noReturnNoParam=()->{
            System.out.println("OK!");
        };
        noReturnNoParam.method();

        /**
         * 有一个参数无返回值
         */
        NoReturnOneParam noReturnOneParam=(int a)->{
            System.out.println("有一个参数无返回值"+a);
        };
        noReturnOneParam.method(10);

        /**
         * 多参数无返回值
         */
        NoReturnMultipleParam noReturnMultipleParam=(int a,int b)->{
            System.out.println("多参数无返回值,参数1:"+a+" ,参数2:"+b);
        };
        noReturnMultipleParam.method(11,22);

        ReturnNoParam returnNoParam=()->{
            System.out.println("无参数的但有返回值");
            return 912;
        };
        System.out.println(returnNoParam.method());


        /**
         * 有一个参数有返回值
         */
        ReturnOneParam returnOneParam=(int a)->{
            System.out.println("有一个参数的有返回值");
            return a;
        };
        System.out.println(returnOneParam.method(34));

        /**
         * 多参有返回值
         */
        ReturnMultiParam returnMultiParam=(int a,int b)->{
            return a+b;
        };
        System.out.println(returnMultiParam.method(45,45));

    }
    
}
```



####  