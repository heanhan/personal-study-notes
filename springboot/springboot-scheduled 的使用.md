### springboot-scheduled 的定时任务的使用方式

scheduled 的是 sprinboot 已经默认集成和哦阿德定时任务，所以使用 scheduled 定时任务不需要引用额外坐标

只需要在启动类上添加注解  @EnableScheduling 即可

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

/**
 * @author zhaojh0912
 */

@EnableScheduling
@SpringBootApplication
public class SpringbootScheduledApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootScheduledApplication.class, args);
    }

}
```

添加注解后 springboot 自动导入相应的注解包，springboot 内部会对原始配置定时任务添加对应的配置文件，创建定时任务类，并在该类上添加注解@Component 将该类纳入 spring bean 中管理。并在定时任务类的方法中 添加注解@Scheduled 配置 cron 定时属性

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

/**
 * @Author :zhaojh0912
 * @Date : 2021/2/10 10:43 上午
 * @Version : 1.0
 * @Description :定时任务  使用cron 表达式
 **/

@Slf4j
@Component
public class ScheduledTask {

    @Scheduled(cron = "0 0/2 8-20 * * ?")
    public void taskOne(){
      log.info("定时任务Cron 表达式任务执行..."+ LocalDateTime.now());

    }
}
```

此时简单的定时任务创建完成。

### 介绍 cron 定时任务的执行表达式。

```
cron 属性
cron 属性值是一个 String 类型的时间表达式，各个部分的含义如下
Seconds 可出现
```

