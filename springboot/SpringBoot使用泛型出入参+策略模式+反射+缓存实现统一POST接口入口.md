 

简介
--

某些情况下需要统一入口，如：提供给第三方调用的接口等。减少接口对接时的复杂性。

### 代码实现

1.  GenericController.java  
    统一入口，通过bean name进行调用service层invoke方法

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;

import java.lang.reflect.InvocationTargetException;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("api")
public class GenericController {

    @PostMapping("/{serviceName}")
    @SuppressWarnings("unchecked")
    public <T> Object invokeService(@PathVariable String serviceName, @RequestBody(required = false) String requestBody) throws JsonProcessingException {
//        log.info("{}接口入参：{}", serviceName, requestBody);
        GenericService<T> genericService = (GenericService<T>) ApplicationContextUtils.getBean(serviceName);
        //通过缓存获取对应 GenericService 实现类 invoke 函数的具体泛型入参。
        Class<T> requestType = (Class<T>) GenericServiceTypeCache.getRequestType(serviceName);
        // 将 JSON 字符串请求参数转换为具体的类型
        T reqObject = JacksonUtils.JSON.readValue(requestBody, requestType);
        try {
            return R.ok(genericService.invoke(reqObject));
        } catch (Exception e) {
            String message = e.getMessage();
            log.error(e.getMessage(), e);
            return R.err(message);
        }
    }

}
```

2.  GenericService.java

```java
/**
 * 通用方法接口
 * @since 2024-07-16
 */
public interface GenericService<T> {
    Object invoke(T req);
}
```

3.  UserGenericServiceImpl.java  
    实现通用接口用户service层类

```java
public class UserGenericServiceImpl implements GenericService<UserDTO> {

    @Override
    public User invoke(UserDTO dto) {
        log.info("UserDTO:{}", dto);
        User user = new User();
        user.setId(1L);
        user.setName("Meta39");
        return user;
    }

}
```

4.  HelloWorldGenericServiceImpl.java  
    实现通用接口打招呼service层类

```java
@Slf4j
@Service("helloWorld")
public class HelloWorldGenericServiceImpl implements GenericService<GenericServiceDTO> {

    @Override
    public String invoke(GenericServiceDTO dto) {
        log.info("GenericServiceDto: {}", dto);
        return "Hello World";
    }

}
```

5.  GenericServiceTypeCache.java  
    通过 HashMap 缓存存储GenericService实现类的invoke函数的泛型请求参数

```java
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.HashMap;
import java.util.Map;

/**
 * 通过 HashMap 缓存存储GenericService实现类的invoke函数的泛型请求参数
 */
@Component
public class GenericServiceTypeCache implements ApplicationRunner {
    /**
     * 只能在启动的时候 put，运行的时候 get。不能在运行的时候 put，因为 HashMap 不是线程安全的。
     */
    private static final Map<String, Class<?>> typeCache = new HashMap<>();

    @Override
    public void run(ApplicationArguments args) throws Exception {
        Map<String, GenericService> beans = ApplicationContextUtils.getBeansOfType(GenericService.class);
        //循环map，forEach(key,value) 是最现代的方式，使用起来简洁明了。也可以用 for (Map.Entry<String, IWebService> entry : beans.entrySet()){}。
        beans.forEach((bean, type) -> {
            // AopProxyUtils.ultimateTargetClass 解决Spring Boot 使用 @Transactional 事务注解的问题。
            Class<?> beanClass = AopProxyUtils.ultimateTargetClass(type);
            // 获取 GenericService 实现类的泛型类型
            Type[] genericInterfaces = beanClass.getGenericInterfaces();
            for (Type genericInterface : genericInterfaces) {
                if (genericInterface instanceof ParameterizedType parameterizedType) {
                    Type[] actualTypeArguments = parameterizedType.getActualTypeArguments();
                    if (actualTypeArguments.length > 0) {
                        Class<?> parameterType = (Class<?>) actualTypeArguments[0];
                        //把泛型入参放入缓存。防止每次请求都通过反射获取入参，影响程序性能。
                        typeCache.put(bean, parameterType);
                    }
                }
            }
        });
    }

    /**
     * 通过缓存获取 GenericService 实现类 invoke 函数的 泛型入参
     *
     * @param serviceName GenericService 实现类的 bean name
     */
    public static Class<?> getRequestType(String serviceName) {
        return typeCache.get(serviceName);
    }

}
```

6.  GenericServiceDto.java  
    第1个数据传输对象看看泛型入参是否可用

```java
@Data
public class GenericServiceDTO {
    private String name;
}
```

7.  UserDTO.java  
    第2个数据传输对象看看泛型入参是否可用

```java
@Data
public class UserDTO {
    private Integer id;
}
```

8.  ApplicationContextUtils.java  
    获取 bean 工具类

```java
/**
 * 获取Bean
 */
@Component
public class ApplicationContextUtils implements ApplicationContextAware {
    //构造函数私有化，防止其它人实例化该对象
    private ApplicationContextUtils() {
    }

    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        ApplicationContextUtils.applicationContext = applicationContext;
    }

    //通过name获取 Bean.（推荐，因为bean的name是唯一的，出现重名的bean启动会报错。）
    public static Object getBean(String name) {
        return applicationContext.getBean(name);
    }

    //通过class获取Bean.（确保bean的name不会重复。因为可能会出现在不同包的同名bean导致获取到2个实例）
    public static <T> T getBean(Class<T> clazz) {
        return applicationContext.getBean(clazz);
    }

    //通过name,以及Clazz返回指定的Bean（这个是最稳妥的）
    public static <T> T getBean(String name, Class<T> clazz) {
        return applicationContext.getBean(name, clazz);
    }

    public static String[] getBeanNamesForType(Class<?> type) {
        return applicationContext.getBeanNamesForType(type);
    }

	public static <T> Map<String, T> getBeansOfType(Class<T> clazz) {
        return applicationContext.getBeansOfType(clazz);
    }

}
```

9.  R.java

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * 统一返回类
 * 创建日期：2024-08-09
 */
@Data
@NoArgsConstructor(access = lombok.AccessLevel.PRIVATE)
@AllArgsConstructor(access = lombok.AccessLevel.PRIVATE)
public class R<T> {
    private Integer code;
    private String message;
    private T data;

    public static <T> R<T> ok() {
        return ok(null);
    }

    public static <T> R<T> ok(T data) {
        return new R<>(1, "success", data);
    }

    public static <T> R<T> err(String message) {
        return err(0, message);
    }

    public static <T> R<T> err(Integer code, String message) {
        return new R<>(code, message, null);
    }

}
```

10.JacksonUtils.java

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.dataformat.xml.XmlMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

/**
 * jackson 工具类
 */
public abstract class JacksonUtils {
    public static final ObjectMapper JSON = new ObjectMapper();
    public static final ObjectMapper XML = new XmlMapper();

    static {
        // json 配置
        JSON.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        JSON.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);
        JSON.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
        JSON.registerModule(new JavaTimeModule());//处理java8新日期时间类型
        // xml 配置
        XML.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        XML.disable(SerializationFeature.FAIL_ON_EMPTY_BEANS);
        XML.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
        XML.registerModule(new JavaTimeModule());//处理java8新日期时间类型
    }

}
```

### 测试

#### helloWorld

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/42feb7d221cb47e8b6f2fa3266dffa45.png)  
[控制台输出](https://so.csdn.net/so/search?q=%E6%8E%A7%E5%88%B6%E5%8F%B0%E8%BE%93%E5%87%BA&spm=1001.2101.3001.7020)

```shell
2024-07-31 23:22:47.204  INFO 10348 --- [spring-boot3-demo] [omcat-handler-1] c.f.s.s.i.HelloWorldGenericServiceImpl   : GenericServiceDto: GenericServiceDTO(name=哈哈哈哈哈)
2024-07-31 23:22:47.212  INFO 10348 --- [spring-boot3-demo] [omcat-handler-1] c.f.springboot3demo.filter.GlobalFilter  : 请求内容:
method: POST
uri: /api/helloWorld
request: { "name":"哈哈哈哈哈" }
2024-07-31 23:22:47.213  INFO 10348 --- [spring-boot3-demo] [omcat-handler-1] c.f.springboot3demo.filter.GlobalFilter  : 响应内容:
status: 200
response: Hello World
```

#### user

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5931fb59b34946f3a342d84e0bb84a12.png)  
控制台输出

```shell
2024-07-31 23:24:46.199  INFO 10348 --- [spring-boot3-demo] [omcat-handler-4] c.f.s.s.impl.UserGenericServiceImpl      : UserDTO:UserDTO(id=1)
2024-07-31 23:24:46.213  INFO 10348 --- [spring-boot3-demo] [omcat-handler-4] c.f.springboot3demo.filter.GlobalFilter  : 请求内容:
method: POST
uri: /api/user
request: { "id":1 }
2024-07-31 23:24:46.213  INFO 10348 --- [spring-boot3-demo] [omcat-handler-4] c.f.springboot3demo.filter.GlobalFilter  : 响应内容:
status: 200
response: {"id":1,"name":"Meta39"}
```

注意事项
----

这种方式实现的统一入口，暂时发现一个弊端，没法使用Spring validation 参数校验框架，否则会抛异常。但是可以通过Spring Assert在代码里判断。

```java
@Slf4j
@Service("helloWorld")
public class HelloWorldGenericServiceImpl implements GenericService<GenericServiceDTO> {

    @Override
    public String invoke(GenericServiceDTO dto) {
        log.info("GenericServiceDto: {}", dto);
        //如下所示
        Assert.notNull(dto, "请求体不能为空");
        return "Hello World";
    }

}
```

本文转自 <https://blog.csdn.net/weixin_43933728/article/details/140834485>，如有侵权，请联系删除。