### springboot--quartz 的使用

#### 使用的技术要求

Springboot 2.4.0 +mybatis+druid+mysql8.0+quartz

#### Quartz 核心的

```
1、Job：Job为作业的接口，为任务调度的对象。

2、JobDetail：用来描述Job实现类及其它相关的静态信息，如Job名字、关联监听器等信息。

3、Trigger：触发器，用于定义任务调度的时间规则，其中CronTrigger是使用cron表达式的触发器，比较常用。cron表达式不太了解的可以自行百度，在这里不多说。

4、Scheduler：任务调度器，是实际执行任务调度的控制器。每个Scheduler都存有JobDetail和Trigger的注册，一个Scheduler中可以注册多个JobDetail和多个Trigger。
```

#### Quartz 的使用方式

##### 1、普通的使用 quartz



##### 2、使用数据的方式使用 quartz 

```xml
<!--Quartz 定时调度依赖-->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
<!--mybatis 插件-->
<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-starter</artifactId>
  <version>2.1.2</version>
</dependency>
<!--mysql 的依赖-->
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <scope>runtime</scope>
</dependency>
<!--Druid 依赖-->
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>druid-spring-boot-starter</artifactId>
  <version>1.1.23</version>
</dependency>
```

application.yml 配置文件

```yml
##微服务的基本信息
server:
  port: 9502
spring:
  application:
    name: springboot-quartz
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/springboot_gather?useSSl=ture&serverTimezone=UTC&useUnicode=true&characterEncoding=UTF-8
    username: root
    password: Root123456
    druid:
      #连接池属性
      initial-size: 15
      max-active: 100
      min-idle: 15
      max-wait: 60000
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      test-on-borrow: false
      test-on-return: false
      test-while-idle: true
      validation-query: SELECT 1
      validation-query-timeout: 1000
      keep-alive: true
      remove-abandoned: true
      remove-abandoned-timeout: 180
      log-abandoned: true
      pool-prepared-statements: true
      max-pool-prepared-statement-per-connection-size: 20
      filters: stat,wall,slf4j
      use-global-data-source-stat: true
      maxOpenPreparedStatements: 100
      connect-properties.mergeSql: true
      connect-properties.slowSqlMillis: 5000

      # 配置DruidStatFilter
      web-stat-filter:
        enabled: true
        url-pattern: "/*"
        exclusions: "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*"
      # 配置DruidStatViewServlet
      stat-view-servlet:
        url-pattern: "/druid/*"
        # IP白名单(没有配置或者为空，则允许所有访问)
        allow: 127.0.0.1
        # IP黑名单 (存在共同时，deny优先于allow)
        deny: 192.168.0.1
        #  禁用HTML页面上的“Reset All”功能
        reset-enable: false
        # 登录名
        login-username: admin
        # 登录密码
        login-password: 123456
        # 新版需要配置这个属性才能访问监控页面
        enabled: true

# MyBatis配置
mybatis:
  # 文件扫描映射文件
  mapper-locations: classpath:mapper/*.xml
  # 配置别名扫描的包
  type-aliases-package: com.example.quartz.entity # 该包下的所有实体在xml文件中就可以直接使用类名了
  # 开启驼峰映射
  configuration:
    map-underscore-to-camel-case: true


```

创建数据库表

``` sql
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
-- Table structure for cron
-- ----------------------------
DROP TABLE IF EXISTS `cron`;
CREATE TABLE `cron`  (
  `id` int NOT NULL AUTO_INCREMENT,
  `cron` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of cron
-- ----------------------------
INSERT INTO `cron` VALUES (1, '0/5 * * * * ? *');
```

