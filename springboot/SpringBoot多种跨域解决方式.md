### SpringBoot多种跨域解决方式

#### 1、垮域怎么理解

##### 跨域是什么

```
垮域是指不同的域名的相互访问，这是浏览器的同源策略决定的，是浏览器对 javascript 施加的安全策略，防止恶意文件的破坏。
```

##### 同源策略 

```
同源策略是一种约定，它是浏览器最核心的也是最基本的安全策略，如果缺少同源策略，则浏览器的正常功能可能会受到影响。所谓同源就是说它的协议、域名、端口完全一致，有一个不一样就会造成跨域问题。
```

##### 垮域原理

- ​	跨域请求能正常发出去，服务器能接受到请求并正常的返回结果，只是结果被拦截。
- 跨域只能存在浏览器，不存在其他平台，比如安卓 、java、ios 等平台。
- 之所以会发生跨域的是因为收到浏览器的同源策略协议的限制，同源策略要求源相同才能正常的通信，即协议、域名、端口号完全一致。

URL

```
统一资源定位符，它是 www 的统一资源定位标志，也就是说我们的网络地址，她的一般格式为：协议类型://服务器地址:端口号/路径。这就是我们说的跨域中的域。
```

#### 2、后端 Springboot 解决跨域

##### 跨域技术 CORS 

cors 是一个 W3C 标准，全称是“跨域资源共享（Cross-Origin Resource Sharing ）”。它允许浏览器向跨域资源服务器，发送 XMLHttpRequest 请求，从而克服了 AJAX 只能同源使用的限制。

**方法一: jsonp**

```
这里不再讲解使用jsonp的方式来解决跨域，因为jsonp方式只能通过get请求方式来传递参数，而且有一些不便之处。
JSOONP的解决跨域的原理  是借用js可以进行远程资源访问，JSONP 就是把返回的结果伪装成js脚本，“骗过”浏览器。
```

**方法二: 注解 @CrossOrigin**

```java
1、前提 spring 4.2版本以上，这个注解可以实现方法级别的细粒度的跨域控制。
我们可以在类或者方添加该注解，如果在类上添加该注解，该类下的所有接口都可以通过跨域访问，如果在方法上添加注解，那么仅仅只限于加注解的方法可以访问。

//@CrossOrigin  表示所有的URL均可访问此资源
@CrossOrigin(origins = "http://127.0.0.1:8093")//表示只允许这一个url可以跨域访问这个controller
@RestController
@RequestMapping("/testCorss")
public class CorssOriginController {

    //可以对方法运用该注解
    //@CrossOrigin(origins = "http://127.0.0.1:8093")
    @GetMapping("/getString")
    public String getString(){
        return "跨域成功！";
    }

}

代码说明：@CrossOrigin这个注解用起来很方便，这个可以用在方法上，也可以用在类上。如果你不设置他的value属性，或者是origins属性，就默认是可以允许所有的URL/域访问。
  1、value属性可以设置多个URL。
  2、origins属性也可以设置多个URL。
  3、maxAge属性指定了准备响应前的缓存持续的最大时间。就是探测请求的有效期。
  4、allowCredentials属性表示用户是否可以发送、处理 cookie。默认为false
  5、allowedHeaders 属性表示允许的请求头部有哪些。
  6、methods 属性表示允许请求的方法，默认get，post，head。
```

**方法三: 实现WebMvcConfigurer**

```
1、通过实现WebConfigure接口，实现接口中的方法，addCorsMapping()方法来实现跨域。
`【补充：WebMvcConfigure、与WebMvcConfigurerAdapter 接口区别
    
    1.接口WebConfigure ：
        Springboot 2.0 以后出现替代WebMvcConfigureradapter接口实现添加添加自定义拦截器，消息转换器等。
    
    2.接口WebMvcConfigurerAdapter：
        springboot 1.5 ，在springboot 2.0以后被标记 @Deprecated 被废弃。
        
    3.接口 WebMvcConfigurationSupport：
        WebMvcConfigurer中有的方法，此类中全都存在。可完全替代WebMvcConfigurer~~~~
        
】`
```

**方法三：通过实现Filter过滤器**

```java
1、拦截器的在所有访问到达前进行对响应体进行设置。

2、步骤：

通过实现Filter 接口 ，重写doFilter方法。将响应ServletResponse 转换为HttpServletResponse 对象 然后设置响应体的信息
@Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {

        /**
         * 实现步骤：
         *  1.将ServletResponse 转换成 HttpServletResponses对象，
         *  2.通过HttpServletResponse 添加访问头信息
         */
        HttpServletResponse resp=(HttpServletResponse)response;
        resp.addHeader("Access-Control-Allow-Credentials", "true");
        resp.addHeader("Access-Control-Allow-Origin", "*");
        resp.addHeader("Access-Control-Allow-Methods", "GET, POST, DELETE, PUT");
        resp.addHeader("Access-Control-Allow-Headers", "Content-Type,X-CAF-Authorization-Token,sessionToken,X-TOKEN");
        if (((HttpServletRequest) request).getMethod().equals("OPTIONS")) {
            //判断请求方法  是否为 OPTIONS 如果是进行拦截，拒绝放问
            return;
        }
        chain.doFilter(request, response);

    }

```

**方法四：使用Nginx 进行跨域**

（如果我们在项目中使用了Nginx，可以在Nginx中添加以下的配置来解决跨域）

```
1、
`location / {
            add_header Access-Control-Allow-Origin *;
            add_header Access-Control-Allow-Headers X-Requested-With;
            add_header Access-Control-Allow-Methods GET,POST,PUT,DELETE,OPTIONS;
         
            if ($request_method = 'OPTIONS') {
              return 204;
            }
         }`
```