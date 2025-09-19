### spring security的学习笔记

spring security 的权限管理，核心就是一个拦截器，对请求进行的认证和授权，

认证：就是用户的登录，告诉系统当前是谁进行登陆了系统。

授权：对需要登录的用户标记该用户可以做哪些操作，规定用户的行为。

一个请求进行访问。首先经过 spring security的 WebSecurityConfigurerAdapter 的核心适配器中。

这个适配器会去调取各种拦截器对请求进行过滤。

1、判断这个请求是否为忽略的资源

2、判断是否登录。【默认的登录逻辑，使用UsernamePasswordAuthenticationFilter 登陆过滤器，进行登录处理认证，AuthenticationProvider ,也可以使用自定义的登录逻辑，对登录进行判断 ，这里有两个处理器需要实现，登录成功处理器，AuthenticationSuccessHandler 和登录失败处理器AuthenticationFailureHandler 】

3、是否登录退出的处理器