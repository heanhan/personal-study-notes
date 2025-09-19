### Sharding-JDBC分库分表简单示例

#### 1. 简介

　　Sharding是一个简单的分库分表中间件，它不需要依赖于其他的服务，即可快速应用在实际项目的分库分表策略中。

#### 2. 初始化数据库（db0、db1、db2）

``` mysql
#创建数据库db0
CREATE DATABASE IF NOT EXISTS `db0` DEFAULT CHARACTER SET utf8mb4;

USE `db0`;

DROP TABLE IF EXISTS `t_user_0`;
CREATE TABLE `t_user_0` (
  `id` int(11) NOT NULL,
  `username` varchar(255) DEFAULT NULL,
  `org_code` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

DROP TABLE IF EXISTS `t_user_1`;
CREATE TABLE `t_user_1` (
  `id` int(11) NOT NULL,
  `username` varchar(255) DEFAULT NULL,
  `org_code` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

DROP TABLE IF EXISTS `t_user_2`;
CREATE TABLE `t_user_2` (
  `id` int(11) NOT NULL,
  `username` varchar(255) DEFAULT NULL,
  `org_code` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

#创建数据库db1
CREATE DATABASE IF NOT EXISTS `db1` DEFAULT CHARACTER SET utf8 ;

USE `db1`;

DROP TABLE IF EXISTS `t_user_0`;
CREATE TABLE `t_user_0` (
  `id` int(11) NOT NULL,
  `username` varchar(255) DEFAULT NULL,
  `org_code` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

DROP TABLE IF EXISTS `t_user_1`;
CREATE TABLE `t_user_1` (
  `id` int(11) NOT NULL,
  `username` varchar(255) DEFAULT NULL,
  `org_code` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

DROP TABLE IF EXISTS `t_user_2`;
CREATE TABLE `t_user_2` (
  `id` int(11) NOT NULL,
  `username` varchar(255) DEFAULT NULL,
  `org_code` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

#创建数据库db2
CREATE DATABASE IF NOT EXISTS `db2` DEFAULT CHARACTER SET utf8;

USE `db2`;

DROP TABLE IF EXISTS `t_user_0`;
CREATE TABLE `t_user_0` (
  `id` int(11) NOT NULL,
  `username` varchar(255) DEFAULT NULL,
  `org_code` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

DROP TABLE IF EXISTS `t_user_1`;
CREATE TABLE `t_user_1` (
  `id` int(11) NOT NULL,
  `username` varchar(255) DEFAULT NULL,
  `org_code` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

DROP TABLE IF EXISTS `t_user_2`;
CREATE TABLE `t_user_2` (
  `id` int(11) NOT NULL,
  `username` varchar(255) DEFAULT NULL,
  `org_code` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 3. 使用 idea 搭建工程

- 修改pom.xml

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  	<modelVersion>4.0.0</modelVersion>
  	<parent>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-parent</artifactId>
          <version>2.4.0</version>
          <relativePath/> <!-- lookup parent from repository -->
    </parent>
  	<groupId>com.example</groupId>
  	<artifactId>springboot-sharding-jdbc</artifactId>
  	<version>0.0.1-SNAPSHOT</version>
  	<packaging>war</packaging>
  	<name>springboot-sharding-jdbc</name>
  	<description>springboot-sharding-jdbc project for Spring Boot</description>
  
  	<properties>
  		<java.version>1.8</java.version>
  		<maven-jar-plugin.version>3.1.1</maven-jar-plugin.version>
  		<mybatis-plus.version>3.3.1</mybatis-plus.version>
  		<sharding-jdbc.version>3.1.0</sharding-jdbc.version>
  	</properties>
  
  	<dependencies>
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-web</artifactId>
  		</dependency>
  
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-tomcat</artifactId>
  			<scope>provided</scope>
  		</dependency>
  		<dependency>
  			<groupId>org.springframework.boot</groupId>
  			<artifactId>spring-boot-starter-test</artifactId>
  			<scope>test</scope>
  		</dependency>
  
  		<dependency>
  			<groupId>mysql</groupId>
  			<artifactId>mysql-connector-java</artifactId>
  			<scope>runtime</scope>
  		</dependency>
  		<dependency>
  			<groupId>com.baomidou</groupId>
  			<artifactId>mybatis-plus-boot-starter</artifactId>
  			<version>${mybatis-plus.version}</version>
  		</dependency>
  		<dependency>
  			<groupId>io.shardingsphere</groupId>
  			<artifactId>sharding-jdbc-spring-boot-starter</artifactId>
  			<version>${sharding-jdbc.version}</version>
  		</dependency>
  		<dependency>
  			<groupId>io.shardingsphere</groupId>
  			<artifactId>sharding-jdbc-spring-namespace</artifactId>
  			<version>${sharding-jdbc.version}</version>
  		</dependency>
  
  	</dependencies>
  
  	<build>
  		<plugins>
  			<plugin>
  				<groupId>org.springframework.boot</groupId>
  				<artifactId>spring-boot-maven-plugin</artifactId>
  			</plugin>
  		</plugins>
  	</build>
  
  </project>
  
  ```

  - 编写实体类

    ```java
    package com.example.sharding.pojo;
    
    import com.baomidou.mybatisplus.annotation.TableField;
    import com.baomidou.mybatisplus.annotation.TableName;
    import com.baomidou.mybatisplus.extension.activerecord.Model;
    import lombok.Data;
    import lombok.EqualsAndHashCode;
    
    /**
     * @Author :zhaojh0912
     * @Date : 2020/12/9 3:01 下午
     * @Version : 1.0
     * @Description :用户信息
     **/
    
    @Data
    @TableName(value = "t_user")
    @EqualsAndHashCode(callSuper = false)
    public class User extends Model<User> {
    
        private static final long serialVersionUID = 1L;
    
        private int id;
    
        private String username;
        
        @TableField(value = "org_code")
        private int orgCode;
    
    }
    
    ```

    - 编写Mapper

      ```java
          package com.example.sharding.mapper;
      
          import com.baomidou.mybatisplus.core.mapper.BaseMapper;
          import com.example.sharding.pojo.User;
      
          /**
           * @Author :zhaojh0912
           * @Date : 2020/12/9 3:04 下午
           * @Version : 1.0
           * @Description :Too
           **/
      
          public interface UserMapper extends BaseMapper<User> {
          }
      
      
      ```

    * 编写 service

      ```java
      package com.example.sharding.service;
      
      import com.example.sharding.pojo.User;
      
      import java.util.List;
      
      /**
       * @Author :zhaojh0912
       * @Date : 2020/12/9 3:05 下午
       * @Version : 1.0
       * @Description :用户service接口
       **/
      
      public interface UserService {
          /**
           * 查询用户列表
           *
           * @return
           */
          List<User> findList();
      
          /**
           * 保存用户信息
           *
           * @param user
           * @return
           */
          boolean save(User user);
      }
      
      ```

      * 编写 service 的实现类

        ```java
        package com.example.sharding.service.impl;
        
        import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
        import com.example.sharding.mapper.UserMapper;
        import com.example.sharding.pojo.User;
        import com.example.sharding.service.UserService;
        import org.springframework.stereotype.Service;
        
        import java.util.List;
        
        /**
         * @Author :zhaojh0912
         * @Date : 2020/12/9 3:06 下午
         * @Version : 1.0
         * @Description :Too
         **/
        
        @Service
        public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService {
        
            /**
             * 查询用户列表
             *
             * @return
             */
            @Override
            public List<User> findList() {
                return new User().selectAll();
            }
        
            /**
             * 保存用户信息
             *
             * @param user
             * @return
             */
            @Override
            public boolean save(User user) {
                return super.save(user);
            }
        }
        
        ```

      * 编写 controller 

        ```java
        package com.example.sharding.controller;
        
        import com.example.sharding.pojo.User;
        import com.example.sharding.service.UserService;
        import org.springframework.beans.factory.annotation.Autowired;
        import org.springframework.web.bind.annotation.GetMapping;
        import org.springframework.web.bind.annotation.PostMapping;
        import org.springframework.web.bind.annotation.RestController;
        
        import java.util.List;
        
        /**
         * @Author :zhaojh0912
         * @Date : 2020/12/9 3:07 下午
         * @Version : 1.0
         * @Description :Too
         **/
        
        @RestController
        public class UserController {
        
            @Autowired
            private UserService userService;
        
            @PostMapping(value = "save")
            public boolean save(User user) {
                return userService.save(user);
            }
        
            @GetMapping(value = "list")
            public List<User> findList() {
                return userService.findList();
            }
        }
        
        ```

        

      * 启动类

        ```java
        package com.example.sharding;
        
        import org.mybatis.spring.annotation.MapperScan;
        import org.springframework.boot.SpringApplication;
        import org.springframework.boot.autoconfigure.SpringBootApplication;
        
        @MapperScan(value = "com.example.sharding.mapper")
        @SpringBootApplication
        public class SpringbootShardingJdbcApplication {
        
        	public static void main(String[] args) {
        		SpringApplication.run(SpringbootShardingJdbcApplication.class, args);
        	}
        
        }
        
        ```

        * Application.yml 配置文件

          ```yml
          
          spring:
            main:
              allow-bean-definition-overriding: true #允许Bean重复注入，后者覆盖前者
            application:
              name: springboot-sharding-jdbc
          sharding:
            jdbc:
              datasource:
                names: db0,db1,db2
                db0:
                  type: com.zaxxer.hikari.HikariDataSource
                  driver-class-name: com.mysql.cj.jdbc.Driver
                  jdbc-url: jdbc:mysql://localhost:3306/db0?useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC
                  username: root
                  password: root
                db1:
                  type: com.zaxxer.hikari.HikariDataSource
                  driver-class-name: com.mysql.cj.jdbc.Driver
                  jdbc-url: jdbc:mysql://localhost:3306/db1?useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC
                  username: root
                  password: root
                db2:
                  type: com.zaxxer.hikari.HikariDataSource
                  driver-class-name: com.mysql.cj.jdbc.Driver
                  jdbc-url: jdbc:mysql://localhost:3306/db2?useUnicode=true&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC
                  username: root
                  password: root
              config:
                props:
                  sql.show: true #打印sql
                sharding:
                  default-database-strategy: #默认分库策略
                    inline:
                      sharding-column: id
                      algorithm-expression: db$->{id % 3}
                  tables:
                    t_user:
                      actual-data-nodes: db$->{0..2}.t_user_$->{0..2} #实际节点
                      table-strategy: #分表策略
                        inline:
                          sharding-column: org_code
                          algorithm-expression: t_user_$->{org_code % 3}
          
          ```

          

