### Redis 是如何实现点赞、取消点赞的？

点赞是一个比较高的频率事件，也不是特别重要的记录，使用缓存来存储还是比较合理，另外像排行榜，热议等都可以使用缓存。

**思路**：本文基于 springcloud 用户发起点赞，取消点赞后先存入 Redis 中，再每隔两个小时从 Redis 读取点赞数据写入数据库中做持久化存储。

#### 分为四个部分介绍使用

- Redis 缓存设计及实现

- 数据库设计

- 数据库操作

- 开启定时任务持久化存储到数据库

  ##### 1、Redis 缓存设计及实现

1.1 安装 reids 步骤省略，

1.2Redis 与 Springboot 项目的整合

1.在 pom.xml 中引入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

2.在启动类上添加注释 @EnableCaching

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableSwagger2
@EnableFeignClients(basePackages = "com.solo.coderiver.project.client")
@EnableCaching
public class UserApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserApplication.class, args);
    }
}

```

1.3 编写 Redis 的配置类 RedisConfig

```java
import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.serializer.Jackson2JsonRedisSerializer;

import java.net.UnknownHostException;


@Configuration
public class RedisConfig {

    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")
    public RedisTemplate<String, Object> redisTemplate(
            RedisConnectionFactory redisConnectionFactory)
            throws UnknownHostException {

        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<Object>(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);

        RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(jackson2JsonRedisSerializer);
        template.setValueSerializer(jackson2JsonRedisSerializer);
        template.setHashKeySerializer(jackson2JsonRedisSerializer);
        template.setHashValueSerializer(jackson2JsonRedisSerializer);
        template.afterPropertiesSet();
        return template;
    }


    @Bean
    @ConditionalOnMissingBean(StringRedisTemplate.class)
    public StringRedisTemplate stringRedisTemplate(
            RedisConnectionFactory redisConnectionFactory)
            throws UnknownHostException {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}

```

至此 Redis 在 SpringBoot 项目中的配置已经完成。

1.4 Redis 的数据结构类型

Redis 可以存储键与 5 种不同数据结构类型之间的映射，这 5 种数据结构分别为 String(字符串)、List(列表)、Set(集合)、Hash（散列）、和 Zset（有序集合）。

下面对这五种数据类型做解释

| 结构类型 | 结构存储值                                     | 结构的读写能力                                               |
| -------- | ---------------------------------------------- | ------------------------------------------------------------ |
| String   | 可以是字符串、整数、浮点值                     | 对整个字符串或者字符串的其中一部分执行操作、对象和浮点数执行资增护着自减 |
| List     | 一个链表，链表上的每一个节点都包含了一个字符串 | 从链表的两端推入或者弹出，根据偏移量对链表进行修剪，读取单个或者多个元素，根据数值来查找或者移除元素 |
| Set      |                                                |                                                              |
| Hash     |                                                |                                                              |
| Zset     |                                                |                                                              |

1.5 点赞数据在 Redis 中的存储格式

用 Redis 存储两种数据，一种记录点赞，被点赞，点赞的状态的数据。另一种是每隔用户被点赞了多少次，做个简单的计数。

由于需要记录点赞人和被点赞人，还有点赞状态（点赞，取消点赞），还要固定事件取出 Redis 中所有点赞数据，分析了 Redis 数据格式中 Hash 最合适。

因为 Hash 里的数据都是存在一个键里，可以通过这个键很方便的把所有的点赞数据都取出。这个键里面的数据还可以存成键值对的形式，方便存入点赞人、被点赞人和点赞状态。

设点赞人的 id 为 likedPostId，被点赞人的 id 为 likedUserId ，点赞时状态为 1，取消点赞状态为 0。将点赞人 id 和被点赞人 id 作为键，两个 id 中间用 :: 隔开，点赞状态作为值。

所以如果用户点赞，存储的键为：likedUserId::likedPostId，对应的值为 1 。取消点赞，存储的键为：likedUserId::likedPostId，对应的值为 0 。取数据时把键用 :: 切开就得到了两个 id，也很方便。