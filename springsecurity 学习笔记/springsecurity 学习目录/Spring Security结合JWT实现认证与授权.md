### Spring Security结合JWT实现认证与授权

Spring Security 的默认Filter链

```java
 SecurityContextPersistenceFilter
->HeaderWriterFilter
->LogoutFilter
->UsernamePasswordAuthenticationFilter
->RequestCacheAwareFilter
->SecurityContextHolderAwareRequestFilter
->SessionManagementFilter
->ExceptionTranslationFilter
->FilterSecurityInterceptor
```

这些过滤器按照既定的优先级排列，最终形成一个过滤链，开发人员也可以自定义过滤链，并通过@Order注解来调整自定义过滤链在过滤器中的位置。

我们重点关注 UsernamePasswordAuthenticationFilter。该过滤器用户处理基于表单方式的登录验证，该过滤器默认只有当请求方法为Post，请求页面为 /login 时过滤器才生效。如果想修改默认拦截url ，只需要在刚才介绍的 Spring Security配置类 WebSecurityConfig中配置该过滤器的拦截路径url  .loginProcessingUrl("url");