# Spring Cloud简介

Spring Cloud 构建于 Spring Boot 之上，是微服务系统架构的一站式解决方案，包括**服务发现注册 、配置中心 、消息总线 、负载均衡 、断路器 、数据监控** 等操作。

## 服务注册与发现 Eureka，Zookeeper，Consul

## 负载均衡 Ribbon

* Ribbon 是运行在消费者端的负载均衡器。

    轮询策略：

    - RoundRobinRule：轮询策略。Ribbon 默认采用的策略。若经过一轮轮询没有找到可用的 provider，其最多轮询 10 轮。若最终还没有找到，则返回 null。
    - RandomRule: 随机策略，从所有可用的 provider 中随机选择一个。
    - RetryRule: 重试策略。先按照 RoundRobinRule 策略获取 provider，若获取失败，则在指定的时限内重试。默认的时限为 500 毫秒。

    更换默认的负载均衡算法，只需要在配置文件中做出修改就行。

    `providerName.ribbon.NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule`
    

    也可以自定义负载均衡算法，只需要实现 IRule 接口，然后修改配置文件或者自定义 Java Config 类。


* Nginx  将所有请求都集中起来，然后再进行负载均衡。

## 服务调用映射 Open Feign

OpenFeign 也是运行在消费者端的，使用 Ribbon 进行负载均衡，所以 OpenFeign 直接内置了 Ribbon。

在导入了 Open Feign 之后我们就可以进行愉快编写 Consumer 端代码了。
```
// 使用 @FeignClient 注解来指定提供者的名字
@FeignClient(value = "eureka-client-provider")public interface TestClient {    
    // 这里一定要注意需要使用的是提供者那端的请求相对路径，这里就相当于映射了    
    @RequestMapping(value = "/provider/xxx",method = RequestMethod.POST) CommonResponse<List<Plan>> getPlans(@RequestBody planGetRequest request);}

```
然后我们在 Controller 就可以像原来调用 Service 层代码一样调用它了。
```
@RestController
public class TestController {
    // 这里就相当于原来自动注入的 Service
    @Autowired
    private TestClient testClient;
    // controller 调用 service 层代码
    @RequestMapping(value = "/test", method = RequestMethod.POST)
    public CommonResponse<List<Plan>> get(@RequestBody 
    planGetRequest request) {
        return testClient.getPlans(request);
    }
}
```

## 熔断和降级 Hystrix
Hystrix 就是一个能进行 **熔断** 和 **降级** 的库，通过使用它能提高整个系统的弹性。
* **熔断** 就是服务雪崩的一种有效解决方案。当指定时间窗内的请求失败率达到设定阈值时，系统将通过 **断路器** 直接将此请求链路断开。

可以使用简单的 @HystrixCommand 注解来标注某个方法，这样 Hystrix 就会使用 断路器 来“包装”这个方法，每当调用时间超过指定时间时(默认为1000ms)，断路器将会中断对这个方法的调用。

也可以对这个注解的很多属性进行设置，比如设置超时时间等：
```
@HystrixCommand(commandProperties = {@HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "1200")})
public List<Xxx> getXxxx() {    // ...省略代码逻辑}
```
* **降级** 是为了更好的用户体验，当一个方法调用异常时，通过执行另一种代码逻辑来给用户友好的回复。这也就对应着 Hystrix 的 后备处理 模式。你可以通过设置 fallbackMethod 来给一个方法设置备用的代码逻辑。
```
// 指定了后备方法调用
@HystrixCommand(fallbackMethod = "getHystrixNews")
@GetMapping("/get/news")
public News getNews(@PathVariable("id") int id) {
    // 调用新闻系统的获取新闻api 代码逻辑省略
}

public News getHystrixNews(@PathVariable("id") int id) {
    // 做服务降级
    // 返回当前人数太多，请稍后查看
}
```

## 服务网关 Zuul
网关是消费者的统一入口，介于客户端与服务器端之间，用于对请求进行鉴权、限流、 路由、监控等功能。

### Zuul 的路由功能

- 简单配置
    ```
    server:  
        port: 9000
    eureka:  
        client:    
            service-url:      
                # 这里只要注册 Eureka 就行了      
                defaultZone: http://localhost:9997/eureka
    ```

- 统一前缀
    ```
    zuul:
        prefix: /zuul
    ```
    原localhost:9000/consumer1/studentInfo/update，配置前缀后需要访问localhost:9000/zuul/consumer1/studentInfo/update

- 路由策略配置

    前面的访问方式(直接使用服务名)，需要将微服务名称暴露给用户，会存在安全性问题。所以，可以自定义路径来替代微服务名称，即自定义路由策略。   
    ```
    zuul:
        routes:
            consumer1: /FrancisQ1/**
            consumer2: /FrancisQ2/**
    ```
    这个时候就可以使用 localhost:9000/zuul/FrancisQ1/studentInfo/update 进行访问了。

- 服务名屏蔽

    在配置完路由策略之后使用微服务名称还是可以访问的，这个时候你需要将服务名屏蔽。
    ```
    zuul:
        ignore-services: "*"
    ```

    - 路径屏蔽
    ```
    zuul:
        ignore-patterns: **/auto/**
    ```
    这样关于 auto 的请求我们就可以过滤掉了。

    注释：

        ** 代表匹配多级任意路径
        *代表匹配一级任意路径

- 敏感请求头屏蔽

    默认情况下，像 Cookie、Set-Cookie 等敏感请求头信息会被 zuul 屏蔽掉，我们可以将这些默认屏蔽去掉，当然，也可以添加要屏蔽的请求头。

### Zuul 的过滤功能

所有请求都经过网关(Zuul)，那么我们可以进行各种过滤，这样我们就能实现 限流，灰度发布，权限控制 等等。

#### 简单实现一个请求时间日志打印

过滤器类型：Pre、Routing、Post。前置Pre就是在请求之前进行过滤，Routing路由过滤器就是我们上面所讲的路由策略，而Post后置过滤器就是在 Response 之前进行过滤的过滤器。

要实现自己定义的 Filter, 我们只需要继承 ZuulFilter, 然后将这个过滤器类以 @Component 注解加入 Spring 容器中就行了。
```
// 加入Spring容器
@Component
public class PreRequestFilter extends ZuulFilter {
    // 返回过滤器类型 这里是前置过滤器
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    // 指定过滤顺序 越小越先执行，这里第一个执行
    // 当然不是只真正第一个 在Zuul内置中有其他过滤器会先执行
    // 那是写死的 比如 SERVLET_DETECTION_FILTER_ORDER = -3
    @Override
    public int filterOrder() { return 0; }

    // 什么时候该进行过滤
    // 这里我们可以进行一些判断，这样我们就可以过滤掉一些不符合规定的请求等等
    @Override
    public boolean shouldFilter() {
        return true;
    }

    // 如果过滤器允许通过则怎么进行处理
    @Override
    public Object run() throws ZuulException {
        // 这里我设置了全局的RequestContext并记录了请求开始时间
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.set("startTime", System.currentTimeMillis());
        return null;
    }
}


@Slf4j
@Component
public class AccessLogFilter extends ZuulFilter {
    // 指定该过滤器的过滤类型
    // 此时是后置过滤器
    @Override
    public String filterType() {
        return FilterConstants.POST_TYPE;
    }

    // SEND_RESPONSE_FILTER_ORDER 是最后一个过滤器
    // 我们此过滤器在它之前执行
    @Override
    public int filterOrder() {
        return FilterConstants.SEND_RESPONSE_FILTER_ORDER - 1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    // 过滤时执行的策略
    @Override
    public Object run() throws ZuulException {
        RequestContext context=RequestContext.getCurrentContext();
        HttpServletRequest request = context.getRequest();
        // 从RequestContext获取原先的开始时间 并通过它计算整个时间间隔
        Long startTime = (Long) context.get("startTime");
        // 这里我可以获取HttpServletRequest来获取URI并且打印出来
        String uri = request.getRequestURI();
        long duration = System.currentTimeMillis() - startTime;
        log.info("uri: {}, duration: {}ms", uri,  duration/100);
        return null;
    }
}
```

#### 限流

* 令牌桶限流

下面通过 Zuul 的前置过滤器来实现一下令牌桶限流。
```
@Component
@Slf4j
public class RouteFilter extends ZuulFilter {
    // 定义一个令牌桶，每秒产生2个令牌，即每秒最多处理2个请求
    private static final RateLimiter RATE_LIMITER = RateLimiter.create(2);
    @Override
    public String filterType() {
        return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
        return -5;
    }

    @Override
    public boolean shouldFilter() {
        RequestContext context = RequestContext.getCurrentContext();
        if(!RATE_LIMITER.tryAcquire()) {
            log.warn("访问量超载");
            // 指定当前请求未通过过滤
            context.setSendZuulResponse(false);
            // 向客户端返回响应码429，请求数量过多
            context.setResponseStatusCode(429);
            return false;
        }
        return true;
    }

    @Override
    public Object run() throws ZuulException {
        log.info("放行");
        return null;
    }
}
```

## 配置中心 Config
一般我们会使用 Bus 消息总线 + Spring Cloud Config 进行配置的动态刷新。

## 消息总线 Bus
Spring Cloud Bus 的作用就是管理和广播分布式系统中的消息，也就是消息引擎系统中的广播模式。

拥有了 Spring Cloud Bus 之后，我们只需要创建一个简单的请求，并且加上 @ResfreshScope 注解就能进行配置的动态修改了。