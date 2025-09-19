随着 IT 技术的发展，越来越多的互联网公司开始采用微服务架构去部署自己的后台服务。在微服务架构中，随着服务数的不断上升，系统的承载能力也越来越强，但是随之带来的问题也开始变得复杂了起来。

日志排查就是其中一个比较重要的问题，在分布式系统集群中，通常一个接口的调用都有可能涉及到好多个项目之间的请求，任意一个服务出现了问题，整个链路就可能会出现中断。而在这张场景下想要找到对应的日志匹配关系就会显得非常困难。

例如我们的调用链路如下图所示：


![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e4799059ffc843bdbb10e75c32a7418a~tplv-k3u1fbpfcp-watermark.image?)

假设请求在执行到支付服务的过程中出现了故障，那么错误日志会记录在支付服务当中，但是这一次请求所关联到的订单处理业务以及库存处理业务的日志，都散落在了订单服务和库存服务中了，那我们该如何将这些错误都关联起来呢？

别急，其实这种问题在业界已经有一套通用的解决方案的，那就是定制一套**全链路跟踪唯一 id**。下边，让我们通过一个实战来了解如何实现全链路日志跟踪唯一 id。

在开始实现这套全链路日志跟踪唯一 id 的方案之前，我们需要先约定好一套环境配置。

我们所模拟的是一个简单的 SpringBoot+Dubbo 类型的分布式服务环境，并没有加入过多复杂的中间件以及网关服务。

## 基本架构分析

由于我们采用的是这种 SpringBoot+Dubbo 类型的搭配来建立自己的服务架构，所以整体从设计上来说，请求的流程应该是如下图所示的：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a6f97819d15745b99e602af434866907~tplv-k3u1fbpfcp-watermark.image?)

因此我们在设计这个链路 id 时，需要首先考虑，如何从 HTTP 请求抵达业务系统的时候，就立马赋予一个链路 id（下边我们统称为 traceId ）。

## 在 Web 层拦截请求

首先，我们需要在流量的入口处，也就是 HTTP 请求的控制器层进行拦截，这部位的拦截可以在 SpringBoot 的过滤器中进行加入，具体是通过定义一个简单的 Filter 组件来实现的：

```java
package 并发编程12.consumer.interceptor;

import dubbo.constants.RequestContentConstants;
import dubbo.context.CommonRequest;
import dubbo.context.CommonRequestContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.UUID;

/**
 * 渠道过滤器
 *
 *  @Author idea
 *  @Date  created in 2:50 下午 2022/7/7
 */
public class ChannelFilter implements Filter {

    private static final Logger LOGGER = LoggerFactory.getLogger(ChannelFilter.class);

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest httpServletRequest = (HttpServletRequest) servletRequest;
        Long userId = convertUserId(httpServletRequest);
        CommonRequest commonRequest = new CommonRequest();
        commonRequest.setTraceId(UUID.randomUUID().toString());
        commonRequest.setUserId(userId);
        CommonRequestContext.put(RequestContentConstants.COMMON_REQUEST, commonRequest);
        LOGGER.info("CommonRequest is {}", commonRequest);
        filterChain.doFilter(servletRequest,servletResponse);
    }

    private static Long convertUserId(HttpServletRequest request) {
        String userIdStr = request.getHeader("user-id");
        if (userIdStr == null) {
            return -1L;
        }
        return Long.valueOf(userIdStr);
    }
}
```

在这个过滤器的内部，我们需要定义好一个请求上下文，用于给每次 HTTP 请求都发配一个唯一的 traceId。这里我是这么设计这个请求上下文的，首先我们需要定义好一个请求体，代码如下：

```
package dubbo.context;

import java.io.Serializable;

/**
 *  @Author  idea
 *  @Date  created in 2:51 下午 2022/7/7
 */
public class CommonRequest implements Serializable {

    private static final long serialVersionUID = -784170575887989214L;

    private String traceId;

    private long userId;

    public String getTraceId() {
        return traceId;
    }

    public void setTraceId(String traceId) {
        this.traceId = traceId;
    }

    public long getUserId() {
        return userId;
    }

    public void setUserId(long userId) {
        this.userId = userId;
    }

    @Override
    public String toString() {
        return "CommonRequest{" +
                "traceId='" + traceId + ''' +
                ", userId=" + userId +
                '}';
    }
}
```

在这个请求体中，我们需要在塞入一个唯一的 traceId，以及和这个请求所关联的一些业务数据，这里我使用的是用户 id 来标记。

接着，我们将这个请求体塞入到一个自定义的请求上下文中：

```
package dubbo.context;

import com.alibaba.ttl.TransmittableThreadLocal;

import java.util.HashMap;
import java.util.Map;

/**
 *  @Author  idea
 *  @Date  created in 3:39 下午 2022/7/7
 */
public class CommonRequestContext {

    private static final ThreadLocal<Map<Object,Object>> requestContentMap = new TransmittableThreadLocal<Map<Object, Object>>(){
        @Override
        protected Map<Object, Object> initialValue() {
            return new HashMap<>();
        }

        @Override
        public Map<Object, Object> copy(Map<Object, Object> parentValue) {
            return parentValue != null ? new HashMap<>(parentValue) : null;
        }
    };

    public static void put(Object key,Object value) {
        requestContentMap.get().put(key,value);
    }

    public static Object get(Object key){
        return requestContentMap.get().get(key);
    }

    public static void clear() {
        requestContentMap.remove();
    }

}
```

这个请求上下文中，其实正好用到了我们前边所学习的线程上下文类的一个扩展组件： TransmittableThreadLocal。TransmittableThreadLocal是基于 ThreadLocal 和 InheritableThreadLocal 的基础上进行设计的一款组件，可以将 A 线程的本地变量透传到线程池内部的线程中。

之所以这里没有使用 ThreadLocal和InheritableThreadLocal，是因为它们存在以下缺点：
-   ThreadLocal：父子线程不会传递threadLocal副本到子线程中
-   InheritableThreadLocal：在子线程创建的时候，父线程会把threadLocal 拷贝到子线中（但是线程池的子线程不会频繁创建，就不会传递信息）

而 TransmittableThreadLocal 则正好解决了上述几点中线程池无法传递线程本地副本的问题，在构造类似 Runnable 接口对象时进行初始化。


现在，我们已经设计好了如何在 HTTP 请求进入业务系统时给它发配对应的 traceId 了。接下来我们再来看看，如何在一个节点发起 Dubbo 调用时给它拦截下来，加入对应的 traceId，以及如何在 Dubbo 服务接收到请求的时候，获取到调用该服务的上一个节点的 traceId。

## 给 Dubbo 调用构建链路 id
在发起 Dubbo 调用之前，请求会先通过 Dubbo 的过滤器层，当请求抵达 Dubbo 的新节点时，也会先通过过滤器层，所以发起调用的整体链路，应该是这样子的：
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9fc16ac9bfb480eb1aba3ca5603a6bb~tplv-k3u1fbpfcp-watermark.image?)

搞清楚了调用链路之后，我们可以尝试在两个过滤器中注入和提取 traceID，从而生成整条 traceId 的链路。下边是两个过滤器的实现部分，首先是分组为 CONSUMER 类型的过滤器：

```
package dubbo.filter;

import com.alibaba.fastjson.JSON;
import dubbo.constants.RequestContentConstants;
import dubbo.context.CommonRequest;
import dubbo.context.CommonRequestContext;
import org.apache.dubbo.common.constants.CommonConstants;
import org.apache.dubbo.common.extension.Activate;
import org.apache.dubbo.rpc.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;


/**
 *  @Author  idea
 *  @Date  created in 2:33 下午 2022/7/1
 */
@Activate(group = CommonConstants.CONSUMER)
public class DubboConsumerTraceFilter implements Filter {

    private static final Logger LOGGER = LoggerFactory.getLogger(DubboConsumerTraceFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        Object o = CommonRequestContext.get(RequestContentConstants.COMMON_REQUEST);
        try {
            if (o != null) {
                CommonRequest commonRequest = (CommonRequest) o;
                LOGGER.info("[DubboConsumerTraceFilter] ------> commonRequest is {}",commonRequest);
                //注入到Dubbo的上下文中
                invocation.getAttachments().put(String.valueOf(RequestContentConstants.COMMON_REQUEST), JSON.toJSONString(commonRequest));
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return invoker.invoke(invocation);
    }
}
```
这段代码中的存在这么一段语句：
`invocation.getAttachments().put(...)` 这句代码其实是在往Dubbo的上下文中注入透传的参数内容。在 Dubbo的RpcInvocation 对象中存在一个叫做 `getAttachments`的 Map 集合，专门用于存储在请求发送过程中需要携带的数据，所以我们在实现透传功能的时候，可以利用好该字段。

Dubbo 底层的 RpcInvocation 对象中的 getAttachments 方法的源代码：
```java
public String getAttachment(String key) {
    if (attachments == null) {
        return null;
    }
    Object value = attachments.get(key);
    if (value instanceof String) {
        return (String) value;
    }
    return null;
}
```


由于第一环节的调用链路是 HTTP 请求入口，所以在进入到 Consumer 端的过滤器时，依然可以从 CommonRequestContext 中获取到的 traceId ，接着我们便可以将它注入到 Dubbo 的上下文中，具体注入的内容就是 CommonRequest 对象的 Json 格式字符串。

到此，Consumer 部分的 traceId 注入基本已经完工了，接下来，让我们来看看在分组是 PROVIDER 类型的过滤器实现部分：

```
package dubbo.filter;


import com.alibaba.fastjson.JSON;
import dubbo.constants.RequestContentConstants;
import dubbo.context.CommonRequest;
import dubbo.context.CommonRequestContext;

import org.apache.dubbo.common.constants.CommonConstants;
import org.apache.dubbo.common.extension.Activate;
import org.apache.dubbo.common.utils.StringUtils;
import org.apache.dubbo.rpc.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 *  @Author  idea
 *  @Date  created in 3:37 下午 2022/7/7
 */
@Activate(group = CommonConstants.PROVIDER)
public class DubboProviderTraceFilter implements Filter {

    private static final Logger LOGGER = LoggerFactory.getLogger(DubboProviderTraceFilter.class);

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        try {
            String attachment = invocation.getAttachment(String.valueOf(RequestContentConstants.COMMON_REQUEST));
            if (StringUtils.isNotEmpty(attachment)) {
                CommonRequest commonRequest = JSON.parseObject(attachment,CommonRequest.class);
                LOGGER.info("[DubboConsumerTraceFilter] ------> commonRequest is {}",commonRequest);
                CommonRequestContext.put(RequestContentConstants.COMMON_REQUEST,commonRequest);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return invoker.invoke(invocation);
    }
}
```

在这个过滤器中，请求会优先从 Dubbo 的上下文中提取出 RequestContext 的 JSON 字符串信息，然后再将它放入到 CommonRequestContext 对象中。而提取的方式也是从 Dubbo的RpcInvocation 对象中的 getAttachment 方法中，这部分的具体源代码如下所示：

```java
public String getAttachment(String key) {
    if (attachments == null) {
        return null;
    }
    Object value = attachments.get(key);
    if (value instanceof String) {
        return (String) value;
    }
    return null;
}
```
ps：记得要在 Dubbo 的 SPI 配置中加入这两个过滤器。
到此，Dubbo 的 Provider 端的 id 链路也算是串联好了，此时整体的链路执行流程大致如下图所示：
![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2cd8ece9f0144092b8fb93da4cb600f6~tplv-k3u1fbpfcp-watermark.image?)



最后，让我们启动整个应用，进行链路的调用。然后在控制台查看打印的日志是否正常。

**请求 consumer 节点日志：**
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c32bb2e1990c461881f01904cff1e109~tplv-k3u1fbpfcp-watermark.image?)

**响应处理的 provider 节点日志：**
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/95afad40a3264ee793d19ab5aece51d6~tplv-k3u1fbpfcp-watermark.image?)

其实到这里，这个全链路日志跟踪唯一 id 的基本模型才算基本完成，有了全链路 id 之后，这些日志如果是有做统一采集的话，在进行问题排查的时候会非常方便。例如在 ELK 平台上通过对 traceId 关键字搜索，从而查询出整条调用链路的请求日志信息。


本章节代码：

```
https://gitee.com/IdeaHome_admin/concurrence-programming-lession/tree/master/src/main/java/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B12
```




## 课后小结

在本节中，我们重点介绍了如何实现一个基本的全链路日志跟踪唯一 id 的方案，通过在各个请求传播的关键过滤器中，结合了并发编程技术中的线程上下文技术，来记录每个线程在请求和传播过程中的 traceId 数值。

但在常见的互联网应用中，工作中所使用的技术组件并不仅仅只有本节所介绍的 SpringBoot、Dubbo、线程池这么简单，随着业务的发展，我们还可能会用 RocketMQ 进行消息消费和生产处理。

所以在本节的最后，我打算留下一道实践性的扩展思考题。

## **课后思考**

**本节课思考**

如何实现兼容 RocketMQ 发送和消费的全链路 id 呢？欢迎在评论区给出你们的思路。




**上节课答疑**

修改 parallelStream 底层 ForkJoinPool 线程数的代码已经提交到了 Gitee 上，具体的修改逻辑是，通过自定义一个 forkjoinPool ，接着往这个 pool 里面丢任务即可，例如下边这段代码：

```java
public ForkJoinPool forkJoinPool = new ForkJoinPool(Runtime.getRuntime().availableProcessors()*2);

/**
* 采用了自定义的forkJoinPool
*
*  @param userIdList
*  @return
 */
public List<UserInfoDTO> batchQueryWithParallelV2(List<Long> userIdList) {
    Future<List<UserInfoDTO>> resultFuture = forkJoinPool.submit(() -> {
        List<UserInfoDTO> resultList = new CopyOnWriteArrayList<>();
        userIdList.parallelStream().forEach(userId -> {
            UserInfoDTO userInfoDTO = userQueryService.queryUserInfoWrapper(userId);
            resultList.add(userInfoDTO);
        });
        return resultList;
    });
    try {
        return resultFuture.get();
    } catch (InterruptedException e) {
        e.printStackTrace();
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
    return null;
}
```


