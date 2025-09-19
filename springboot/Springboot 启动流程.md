### Springboot 启动流程

1、从 springboot 的入口函数 mian 所在的类上有重要的注解

@SpringbootApplication，此注解是一个组合注解，包含@SpringbootConfiguration（是打 @SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan ：扫描注解，自动扫描符合条件的组件（@Service、@Compoonent）或者 bean 定义，记载到 IOC 容器中。

springboot 启动流程

1、从 spring.factories 配置文件中加载 EventPublishRunListener 对象，该对象拥有 SimpleApplicationEventMulticaster 属性，即 SpringBoot 启动过程的不同阶段用来发射内置生命周期事件。

2、准备环境变量，包括系统变量、环境变量、命令行参数、默认变量、servlet 相关配置变量。随机值以及配置文件（比如 application.properties）等。

3、控制台打印 springboot 的 banner 标志，

4、根据不同类型环境创建不同的 applicationContext 容器，如果是 servlet 环境创建的就是AnnatationConfigServeletWebServerApplicationContext 容器对象。

5、从 spring.factories 配置文件中加载 FailureAnalyzers 对像，用来做报告 SpringBoot 启动中的异常。

6、为刚创建的容器对象做一些初始化的工作，准备一些容器属性值等，对 ApplicationContext 应用一些相关的后置处理和调用各个 ApplicationContextInitializer 的初始化来执行一些初始化逻辑。

7、刷新容器，这一步至关重要，比如调用 bean factory 的后置处理器，注册 BeanPostProcessor 后置处理器，初始化时间广播和广播事件，初始化剩下的单例 bean 和 SpringBoot 创建内嵌 Tomcat 服务器的呢个重要且复杂的逻辑都在这里实现。

8、执行刷新容器后的后置处理逻辑，注意这里为空方法，

9、调用 ApplicationRunner 和 CommandlindRunner 的 run 方法，我们实现这两个接口可以在容器启动后需要的一些东西，比如加载一些业务数据。

10、报告启动异常，即若启动过程中抛出异常，此时用failuerAnalyzers 来包好异常;

11、最终返回容器对象，这里调用方法没有声明对象来接收。

