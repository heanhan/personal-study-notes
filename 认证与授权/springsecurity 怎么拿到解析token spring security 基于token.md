基于Spring Boot3实现Spring Security6 + JWT + Redis实现登录、token身份认证。

> *   用户从数据库中获取。
> *   使用RESTFul风格的APi进行登录。
> *   使用JWT生成token。
> *   使用Redis进行登录过期判断。
> *   所有的工具类和数据结构在源码中都有。

  

#### 文章目录

*   依赖版本
*   原理
*   代码结构

*   security 配置
*   用户登录、注册controller，用户服务
*   用到的工具类

*   注册 AuthController.register
*   登录

*   1.登录API：AuthController.login
*   2\. 登录过滤器：继承UsernamePasswordAuthenticationFilter
*   3.身份认证：实现AuthenticationProvider
*   4.从数据库中查询用户信息：实现UserDetailsService
*   5\. Security配置: 使用注解@EnableWebSecurity

*   token身份认证

*   1\. token身份认证过滤器： OncePerRequestFilter

*   UserAuthUtils

*   getUserId

*   用户登出

*   实现LogoutSuccessHandler
*   修改Security配置 : YaSecurityConfig

*   下一步的计划
*   参考文章

  

### 依赖版本

*   Spring Boot 3.0.6
*   Spring Security 6.0.3

### 原理

这张图大家已经估计已经看过很多次了。  

![springsecurity 怎么拿到解析token spring security 基于token_jwt](https://s2.51cto.com/images/blog/202406/27151616_667d11c0a4ba38257.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184 "原理")

  

实现登录认证的过程，其实就是对上述的类按照自己的需求进行自定义扩展的过程。具体不多讲了，别的文章里讲得比我透彻。  

_**show you my code.**_

### 代码结构

#### security 配置

![springsecurity 怎么拿到解析token spring security 基于token_spring security_02](https://s2.51cto.com/images/blog/202406/27151616_667d11c0e01ec80421.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184 "在这里插入图片描述")

#### 用户登录、注册controller，用户服务

![springsecurity 怎么拿到解析token spring security 基于token_spring security_03](https://s2.51cto.com/images/blog/202406/27151617_667d11c12a87017495.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184 "在这里插入图片描述")

#### 用到的工具类

![springsecurity 怎么拿到解析token spring security 基于token_ide_04](https://s2.51cto.com/images/blog/202406/27151617_667d11c1826933895.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184 "在这里插入图片描述")

### 注册 AuthController.register

将用户密码使用`BCrypt`加密存储。

登录后复制

```java
@PostMapping("/register")
    @Operation(summary = "register", description = "用户注册")
    public Object register(@RequestBody @Valid UserRegisterDTO userRegisterDTO) {
        YaUser userById = userService.getUserById(userRegisterDTO.getUserId());
        if(Objects.nonNull(userById)){
            return BaseResult.fail("用户id已存在");
        }
        try {
            BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();
            YaUser yaUser = UserRegisterMapper.INSTANCE.registerToUser(userRegisterDTO);
            yaUser.setUserPassword(encoder.encode(userRegisterDTO.getUserPassword()));
            userService.insertUser(yaUser);
            return BaseResult.success("用户注册成功");
        }catch (Exception e){
            return BaseResult.fail("用户注册过程中遇到异常：" + e);
        }
    }
```

### 登录

#### 1.登录API：AuthController.login

我们使用RESTFul风格的API来代替表单进行登录。这个接口只是提供一个Swagger调用登录接口的入口，实际逻辑由Filter控制。  

![springsecurity 怎么拿到解析token spring security 基于token_ide_05](https://s2.51cto.com/images/blog/202406/27151617_667d11c1a2e9d87498.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184 "在这里插入图片描述")

#### 2\. 登录过滤器：继承UsernamePasswordAuthenticationFilter

> 拦截指定的登录请求，交给AuthenticationProvider处理。对Provider返回的登录结果进行处理。

*   通过指定`filterProcessesUrl`,指定登录接口的路径。
*   登录失败，将异常信息返回前端。
*   登录成功，通过`JwtUtils`生成token，放入响应header中。并将`token`和`用户信息(json字符串)`存入Redis中。
*   通过`JwtUtils`生成token设置为永不过期，存入Redis的token过期时间设置为30分钟，以便后边做登录过期的判断。

登录后复制

```java
/**
 * <p>
 *  拦截登陆过滤器
 * </p>
 *
 * @author Ya Shi
 * @since 2024/3/21 16:20
 */
@Slf4j
public class YaLoginFilter extends UsernamePasswordAuthenticationFilter {
    private final RedisUtils redisUtils;

    private final Long expiration;

    public YaLoginFilter(AuthenticationManager authenticationManager, RedisUtils redisUtils, Long expiration) {
        this.expiration = expiration;
        this.redisUtils = redisUtils;
        super.setAuthenticationManager(authenticationManager);
        super.setPostOnly(true);
        super.setFilterProcessesUrl("/auth/login");
        super.setUsernameParameter("userId");
        super.setPasswordParameter("userPassword");
    }

    @SneakyThrows
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {
        log.info("YaLoginFilter authentication start");
        // 数据是通过 RequestBody 传输
        UserLoginDTO user = JSON.parseObject(request.getInputStream(), StandardCharsets.UTF_8, UserLoginDTO.class);

        return super.getAuthenticationManager().authenticate(
                new UsernamePasswordAuthenticationToken(user.getUserId(), user.getUserPassword())
        );
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                            FilterChain chain,
                                            Authentication authResult) {
        log.info("YaLoginFilter authentication success: {}", authResult);
        // 如果验证成功, 就生成Token并返回
        UserDetails userDetails = (UserDetails) authResult.getPrincipal();
        String userId = userDetails.getUsername();
        String token = JwtUtils.generateToken(userId);
        response.setHeader(TOKEN_HEADER, TOKEN_PREFIX + token);
        // 将token存入Redis中
        redisUtils.set(REDIS_KEY_AUTH_TOKEN + userId, token, expiration);
        log.info("YaLoginFilter authentication end");
        // 将UserDetails存入redis中
        redisUtils.set(REDIS_KEY_AUTH_USER_DETAIL + userId, JSON.toJSONString(userDetails), 1, TimeUnit.DAYS);

        ServletUtils.renderResult(response, new BaseResult<>(ResultEnum.SUCCESS.code, "登陆成功"));
        log.info("YaLoginFilter authentication end");
    }

    /**
     * 如果 attemptAuthentication 抛出 AuthenticationException 则会调用这个方法
     */
    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                              AuthenticationException failed) throws IOException {
        log.info("YaLoginFilter authentication failed: {}", failed.getMessage());
        ServletUtils.renderResult(response, new BaseResult<>(ResultEnum.FAILED_UNAUTHORIZED.code, "登陆失败：" + failed.getMessage()));
    }
```

#### 3.身份认证：实现AuthenticationProvider

> 调用UserDetailsService查询用户的账户、权限信息与登录接口输入的账户、密码对比。认证通过则返回用户信息。

登录后复制

```java
/**
 * <p>
 *  自定义认证
 * </p>
 *
 * @author Ya Shi
 * @since 2024/3/21 15:00
 */

@Component
public class YaAuthenticationProvider implements AuthenticationProvider {
    @Autowired
    YaUserDetailService userDetailService;

    @Autowired
    PasswordEncoder passwordEncoder;


    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        // 获取用户输入的用户名和密码
        String username = authentication.getName();
        String password = authentication.getCredentials().toString();
        UserDetails userDetails = userDetailService.loadUserByUsername(username);
        boolean matches = passwordEncoder.matches(password, userDetails.getPassword());
        if(!matches){
            throw new AuthenticationException("User password error."){};
        }
        return new UsernamePasswordAuthenticationToken(userDetails, userDetails.getPassword(), userDetails.getAuthorities());
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

#### 4.从数据库中查询用户信息：实现UserDetailsService

> 从数据库中查询出用户的信息，供AuthenticationProvider登录认证时使用。

*   用户权限这块，目前还没用到，可以忽略。用户鉴权可能后边会单独补上。
*   为什么这里没先从Redis取用户信息？

1.  如果权限或者用户信息变更这里取不到
2.  Redis里不建议存储用户密码。

登录后复制

```java
/**
 * <p>
 *  继承UserDetailsService,实现自定义登陆认证
 * </p>
 *
 * @author Ya Shi
 * @since 2024/3/19 11:32
 */
@Service
public class YaUserDetailService implements UserDetailsService {

    @Autowired
    UserService userService;
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        YaUser user = userService.getUserById(username);
        if(Objects.isNull(user)){
            throw new UsernameNotFoundException("User not Found.");
        }
        List<YaUserRole> roles = userService.listRoleById(username);
        List<GrantedAuthority> authorities = new ArrayList<>(roles.size());
        roles.forEach( role -> authorities.add(new SimpleGrantedAuthority(role.getRoleId())));

        return new User(username, user.getUserPassword(), authorities);
    }
}
```

#### 5\. Security配置: 使用注解@EnableWebSecurity

*   注意：Spring Security 6 配置不再继承adapter`extends WebSecurityConfigurerAdapter`，而是使用`@EnableWebSecurity`。
*   `YaTokenFilter`是token身份认证过滤器，每次请求都会拦截，然后校验请求header中的token，这个下面会讲。
*   配置了身份认证过滤器以后，每个请求都会被拦截，即使是在过滤链中配置了`permitAll()`，还是会返回请求403.

1.  因此，针对匿名请求、静态资源和swagger请求，在`WebSecurityCustomizer`中配置`WebSecurity.ignoring`，相当于直接绕过所有的Filter
2.  针对登录和注册请求，在身份过滤器中额外配置白名单，单独放行。

*   自己学习的过程中，很多文章没有按照代码执行顺序去讲，登录和身份认证也是混着讲的，导致整个登录认证的流程理解起来有些困难。

登录后复制

```java
/**
 * <p>
 *  Spring Security 配置文件
 * </p>
 *
 * @author Ya Shi
 * @since 2024/2/29 11:27
 */
@Configuration
@EnableWebSecurity // 开启网络安全注解
public class YaSecurityConfig {

    @Autowired
    private AuthenticationConfiguration authenticationConfiguration;

    @Autowired
    private RedisUtils redisUtils;

    @Value("${ya-app.auth.jwt.expiration:1800}")
    private Long expiration;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                // 禁用basic明文验证
                .httpBasic().disable()
                // 禁用csrf保护
                .csrf().disable()
                // 禁用session
                .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                // 身份认证过滤器
                .authenticationManager(authenticationManager(authenticationConfiguration))
                .authenticationProvider(new YaAuthenticationProvider())
                .authorizeHttpRequests(authorizeHttpRequests ->
                        authorizeHttpRequests
                                // 允许OPTIONS请求访问
                                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                                // 允许登录/注册接口访问
                                .requestMatchers(HttpMethod.POST, "/auth/login").permitAll()
                                .requestMatchers(HttpMethod.POST, "/auth/register").permitAll()
                                // 允许匿名接口访问
                                .requestMatchers("/anon/**").permitAll()
                                // 允许swagger访问
                                .requestMatchers("/swagger-ui/**").permitAll()
                                .requestMatchers("/doc.html/**").permitAll()
                                .requestMatchers("/v3/api-docs/**").permitAll()
                                .requestMatchers("/webjars/**").permitAll()
                                .anyRequest().authenticated()
                )
                .addFilterAt(new YaLoginFilter(authenticationManager(authenticationConfiguration), redisUtils, expiration), UsernamePasswordAuthenticationFilter.class)
                // 让校验Token的过滤器在身份认证过滤器之前
                .addFilterBefore(new YaTokenFilter(redisUtils, expiration), YaLoginFilter.class)
                // 禁用默认登出页
                .logout().disable();
        return http.build();
    }


    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public WebSecurityCustomizer webSecurityCustomizer() {
        return (web) -> web.ignoring()
                .requestMatchers("/webjars/**")
                .requestMatchers("/swagger-ui/**", "/doc.html/**", "/v3/api-docs/**")
                .requestMatchers("/anon/**");
    }

    /**
     * 使用BCrypt加密密码
     */
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### token身份认证

#### 1\. token身份认证过滤器： OncePerRequestFilter

*   对于注册、登录请求，直接放行。
*   认证失败的几种情况：

1.  未登录： 未携带token
2.  凭证异常： 携带错误token
3.  登录过期： 携带正确的token，但是token在Redis中不存在
4.  账号在别处登录： 携带正确的token，但是token与Redis中的token不一致。

*   token认证成功后，重新设置Redis中的token的有效时间，实现token续期。查询Redis中的用户信息，如果没有，使用UserDetailsService的服务重新查询出信息，存入缓存中。  
    \*调用 `SecurityContextHolder.getContext().setAuthentication()`将用户信息存入Security上下文中，完成身份认证。

登录后复制

```java
/**
 * <p>
 *  每次请求过滤token
 * </p>
 *
 * @author Ya Shi
 * @since 2024/3/21 16:52
 */
@Slf4j
public class YaTokenFilter extends OncePerRequestFilter {

    private final RedisUtils redisUtils;

    private final Long expiration;

    private static final Set<String> WHITE_LIST = Stream.of(
            "/auth/register",
            "/auth/login"
            ).collect(Collectors.toSet());

    public YaTokenFilter(RedisUtils redisUtils, Long expiration) {
        this.redisUtils = redisUtils;
        this.expiration = expiration;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("YaTokenFilter doFilterInternal start");
        final String authorization = request.getHeader(AuthConstants.TOKEN_HEADER);
        log.info("YaTokenFilter ya-auth-token: {}", authorization);

        // 白名单
        if (WHITE_LIST.contains(request.getServletPath())) {
            chain.doFilter(request, response);
            return;
        }

        // 1.请求头中没有携带token
        if (StrUtil.isBlank(authorization)) {
            ServletUtils.renderResult(response, new BaseResult<>(ResultEnum.FAILED_UNAUTHORIZED));
            return;
        }

        // 携带token
        final String token = authorization.replace(AuthConstants.TOKEN_PREFIX, "");
        String userId;
        // 2.提供的token异常
        try {
            userId =  JwtUtils.extractUserId(token);
        }catch (Exception e){
            log.error("YaTokenFilter doFilterInternal 解析jwt异常：{}", e.toString());
            ServletUtils.renderResult(response, new BaseResult<>(ResultEnum.FAILED_UNAUTHORIZED.code, "凭证异常"));
            return;
        }

        String redisToken = redisUtils.getString(AuthConstants.REDIS_KEY_AUTH_TOKEN + userId);
        // 3.token过期
        if(StrUtil.isBlank(redisToken)){
            ServletUtils.renderResult(response, new BaseResult<>(ResultEnum.FAILED_UNAUTHORIZED.code, "登录已过期，请重新登录过期。"));
            return;
        }

        // 4.提供的token是合法的，但是redis中的token又被使用登录功能重新刷新了一下，导致不一致。
        if(!Objects.equals(redisToken, token)){
            ServletUtils.renderResult(response, new BaseResult<>(ResultEnum.FAILED_UNAUTHORIZED.code, "账号在别处登陆。"));
            return;
        }
        // token续期
        redisUtils.set(REDIS_KEY_AUTH_TOKEN + userId, token, expiration);
        // 获取用户信息和权限
        String userDetailStr =  redisUtils.getString(AuthConstants.REDIS_KEY_AUTH_USER_DETAIL + userId);
        UserDetails userDetails;
        if(Objects.isNull(userDetailStr)){
            userDetails = yaUserDetailService().loadUserByUsername(userId);
            redisUtils.set(REDIS_KEY_AUTH_USER_DETAIL + userId, JSON.toJSONString(userDetails), 1, TimeUnit.DAYS);
        }else{
            userDetails = initUser(userDetailStr);
        }
        SecurityContextHolder.getContext().setAuthentication(
                new UsernamePasswordAuthenticationToken(
                        userDetails, userDetails.getPassword(), userDetails.getAuthorities()
                )
        );
        log.info("YaTokenFilter doFilterInternal end");
        chain.doFilter(request, response);
    }

    private YaUserDetailService yaUserDetailService(){
        return SpringContextUtils.getBean(YaUserDetailService.class);
    }

    private User initUser(String userJsonStr){
        JSONObject userJson = JSON.parseObject(userJsonStr);

        String userId = userJson.getString("username");
        JSONArray authArray = userJson.getJSONArray("authorities");

        List<GrantedAuthority> authorities = new ArrayList<>(authArray.size());
        for(int i=0; i< authArray.size();i++){
            JSONObject authObj = authArray.getJSONObject(i);
            authorities.add(new SimpleGrantedAuthority(authObj.getString("authority")));
        }
        return new User(userId, "[PROTECTED]", authorities);
    }

}
```

### UserAuthUtils

> 已经登录的用户，可以从Security的上下文中获取用户的账号、基本信息、权限等。可以将其封装为工具类。因为练手的用户表较为简单，也没有部分、员工、角色、权限等概念，因此仅封装了getUserId做抛砖引玉的作用。可以根据实际使用自己封装更多的方法。

#### getUserId

登录后复制

```java
public static String getUserId() {
        if (Objects.isNull(SecurityContextHolder.getContext().getAuthentication())) {
            return null;
        }
        UserDetails userDetails = (UserDetails) SecurityContextHolder.getContext().getAuthentication().getPrincipal();
        if (Objects.isNull(userDetails)) {
            return null;
        }
        return userDetails.getUsername();
    }
```

### 用户登出

> JWT本身是无状态的，但是我们后端将jwt存到redis里，相当于手动使JWT变得有状态了。那么我们在登出时就需要清空Redis中的jwt。

#### 实现LogoutSuccessHandler

登录后复制

```java
/**
 * <p>
 *  登出成功
 * </p>
 *
 * @author Ya Shi
 * @since 2024/3/28 10:47
 */
@Slf4j
public class YaLogoutSuccessHandler implements LogoutSuccessHandler {
    private final RedisUtils redisUtils;

    public YaLogoutSuccessHandler(RedisUtils redisUtils) {
        this.redisUtils = redisUtils;
    }

    @Override
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {
        final String authorization = request.getHeader(AuthConstants.TOKEN_HEADER);
        // 1.请求头中没有携带token
        if (StrUtil.isBlank(authorization)) {
            ServletUtils.renderResult(response, BaseResult.successWithMessage("没有登录信息，无需退出"));
            return;
        }

        // 携带token
        final String token = authorization.replace(AuthConstants.TOKEN_PREFIX, "");
        String userId;
        // 2.提供的token异常
        try {
            userId =  JwtUtils.extractUserId(token);
        }catch (Exception e){
            log.error("YaLogoutHandler logout 解析jwt异常：{}", e.toString());
            ServletUtils.renderResult(response, new BaseResult<>(ResultEnum.FAILED_UNAUTHORIZED.code, "凭证异常"));
            return;
        }
        // 清空Redis
        redisUtils.delete(REDIS_KEY_AUTH_TOKEN + userId);
        log.info("YaLogoutSuccessHandler onLogoutSuccess");
        ServletUtils.renderResult(response, BaseResult.successWithMessage("退出登录成功"));
    }
}
```

#### 修改Security配置 : YaSecurityConfig

登录后复制

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
	http  ... // 前面的配置忽略
	.logout().logoutUrl("/auth/logout").logoutSuccessHandler(new YaLogoutSuccessHandler(redisUtils));
	return http.build();
}
```

### 下一步的计划

*   用户鉴权
*   排查`permitAll()`失效的问题。
*   做一个练手用的用户中心，提供统一的注册、登录、认证、鉴权服务，供其他的应用调用。
*   把前期已经实现的基础的配置和工具类封装为jar包，供以后的程序使用。

### 参考文章

*    SpringBoot3.0 + SpringSecurity6.0+JWT
*    SpringSecurity (3) SpringBoot + JWT 实现身份认证和权限验证
*    Spring Security 6.0(spring boot 3.0) 下认证配置流程
*    SpringSecurity的PermitAll、WebSecurityCustomizer和授权
*    springsecurity的http.permitall与web.ignoring的区别
