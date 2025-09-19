## （一）Spring Security中的认证与密码编码器

`Spring Security`是一个提供认证(authentication)、授权(authorization)和防御各种Web攻击的框架，它对命令式和反应式应用程序都提供了一流的支持，是保护基于spring的应用程序的事实标准。

**Spring Security中的Authentication(认证)**

`spring security`提供了用于**认证**、**授权**和**保护应用**受到常见的各种恶意攻击的全面支持，同时也提供了与第三方库的集成，并简化了其应用。

`Authentication`(认证) 是指我们以何种方式识别访问特定资源者的身份，常用的方式是要求用户在访问前输入**用户名**和**密码**。一旦执行完了认证操作，我们就确认了访问者的身份，并可以进一步执行授权操作。

**Spring Security中的密码存储**

`Spring Security`的`PasswordEncoder`接口是用来执行密码单向加密后安全存储的一种方式。既然`PasswordEncoder`是单向加密，那么当密码需要反向解密时时就不打算使用它。`PasswordEncoder`的典型使用场景是**存储的密码需要在用户认证时与用户提供的密码进行比对**。