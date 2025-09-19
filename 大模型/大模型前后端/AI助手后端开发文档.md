## AI模型后端文档

项目技术选型

基于spring-data-jpa+ mysql +springsecurity 、jwt、redis 的等技术

项目技术架构

```
src/main/java/com/example/authsystem/
├── config/                # 配置类
│   ├── SecurityConfig.java
│   ├── RedisConfig.java
│   ├── CorsConfig.java
│   └── JpaConfig.java
├── controller/            # 控制器
│   ├── AuthController.java
│   └── UserController.java
├── dto/                   # 数据传输对象
│   ├── LoginDTO.java
│   ├── PhoneLoginDTO.java
│   ├── RegisterDTO.java
│   ├── TokenDTO.java
│   └── UserDTO.java
├── entity/                # 实体类
│   ├── Role.java
│   ├── User.java
│   └── UserRole.java
├── exception/             # 异常处理
│   └── GlobalExceptionHandler.java
├── filter/                # 过滤器
│   └── JwtAuthenticationFilter.java
├── repository/            # JPA仓库
│   ├── RoleRepository.java
│   ├── UserRepository.java
│   └── UserRoleRepository.java
├── security/              # 安全相关
│   ├── JwtTokenProvider.java
│   ├── UserPrincipal.java
│   └── UserDetailsServiceImpl.java
├── service/               # 服务层
│   ├── AuthService.java
│   ├── SmsService.java
│   └── UserService.java
├── util/                  # 工具类
│   ├── RedisUtil.java
│   └── ThreadLocalUtil.java
└── AuthSystemApplication.java
```

### 现在我们已经完成了所有必要的代码实现。这个实现包括：

1. 用户认证和授权

1. JWT token的生成和验证

1. Redis用于存储token和验证码

1. 用户信息的ThreadLocal存储

1. 跨域支持

1. 验证码登录流程

1. Token刷新机制

### 主要功能点：

1. 用户通过手机号和验证码登录

1. 系统验证验证码正确性

1. 如果用户不存在则自动注册

1. 生成access_token和refresh_token

1. access_token存储在Redis中，有效期2小时

1. 实现了JWT认证过滤器

1. 使用ThreadLocal存储用户信息

1. 实现了退出登录功能

1. 支持token刷新

### 使用说明：

1. 登录接口：POST /api/auth/login?phone=手机号&code=验证码

1. 退出接口：POST /api/auth/logout

1. 刷新token接口：POST /api/auth/refresh?refreshToken=刷新令牌

### 注意事项：

1. 需要确保MySQL服务已启动，并创建了名为chats的数据库

1. 需要确保Redis服务已启动

1. 需要根据实际环境修改application.properties中的数据库和Redis配置

1. 验证码的生成和发送功能需要另外实现