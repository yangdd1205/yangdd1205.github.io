---
title: Spring Cloud Hystrix：断路器
tags:
  - Spring Cloud
categories: Spring Cloud
---

<!-- 介绍下什么是断路器，生活中的例子 -->

在分布式系统中，服务之间的调用难免会因为网络原因或者服务自身的问题出现延迟或者故障，如果不及时解决，随着请求不断的增加，最后积压的问题就如洪水决堤，轻则导致自身服务的瘫痪，重则导致整个系统的瘫痪，那么解决怎个问题呢？在日常生活中有个叫断路器的设备，能够在电流过载或发送短路的时候自动切断电路，做到对电源线路及电动机的保护。
{% blockquote 百度百科, 断路器 %}
断路器是指能够关合、承载和开断正常回路条件下的电流并能关合、在规定的时间内承载和开断异常回路条件下的电流的开关装置。可用来分配电能，不频繁地启动异步电动机，对电源线路及电动机等实行保护，当它们发生严重的过载或者短路及欠压等故障时能自动切断电路，其功能相当于熔断器式开关与过欠热继电器等的组合。
{% endblockquote %}
那么在上诉分布式系统存在的问题，是不是也可以用类似的思路来解决呢？

<!-- more -->
## Hystrix

Spring Cloud Hytrix 是 基于 Netflix 的开源框架 Hystrix 做了封装，该框架的目的是通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix 具备了服务降级、服务熔断、线程隔离、请求缓存、请求合并以及服务监控等强大的功能。目前托管在 [GitHub](https://github.com/Netflix/Hystrix) 上。

## 简单使用

现有三个项目如下：
* [server](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-03-hystrix-server)：注册中心，服务名：eureka-service，端口：7001
* [client](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-03-hystrix-client)：服务提供者，服务名：client，端口：8001
* [request-a](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-03-hystrix-request-a)：服务调用者，服务名：request，端口 9001

`client` 对外提供如下服务：
```Java
@RestController
public class HelloController {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String sayHello() throws InterruptedException {
        TimeUnit.SECONDS.sleep(3);
        System.out.println("hello hystrix!");
        return "hello hystrix!";
    }

    @RequestMapping(value = "/hi", method = RequestMethod.GET)
    public String hi(String name) throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        System.out.println("hi " + name + "!");
        return "hi " + name + "!";
    }

    @RequestMapping(value = "/request", method = RequestMethod.GET)
    public String request() throws InterruptedException {
        System.out.println("request...");
        return "request";
    }
}
   
```

在 `request-a` 中，我们先加入 `spring-cloud-starter-hystrix` 的依赖：
```XML
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrixs</artifactId>
</dependency>
```

我们可以通过注解 `@EnableCircuitBreaker` 启动断路器功能。
```Java
@EnableCircuitBreaker // 开启断路器功能
@EnableDiscoveryClient
@SpringBootApplication
public class RequestApplication {

    @Bean
    @LoadBalanced // 自动负载均衡：他的机制是（通过）Application Name去寻找服务发现，然后去做负载均衡策略的
    public RestTemplate restTemplate() {
        return new RestTemplate(httpRequestFactory());
    }

    @Bean
    @ConfigurationProperties(prefix = "custom.http")
    public HttpComponentsClientHttpRequestFactory httpRequestFactory() {
        return new HttpComponentsClientHttpRequestFactory();
    }
   
    
    public static void main(String[] args) {
        SpringApplication.run(RequestApplication.class, args);
    }
}
```

调用 `client` 的服务，可以在具体的方法上，通过注解 `@HystrixCommand` 来指定服务降级方法，指定的降级方法的参数与返回值需和主逻辑方法一样。
```Java
@Service
public class HelloService {

    @Autowired
    private RestTemplate restTemplate;

    @HystrixCommand(fallbackMethod = "callHelloFailback")
    public String callHello() {
        return restTemplate.getForEntity("http://client/hello", String.class).getBody();
    }

    public String callHelloFailback() {
        System.err.println("callHello 执行降级策略");
        return "callHelloFailback";
    }

    @HystrixCommand(fallbackMethod = "handlerFailback", ignoreExceptions = { FileNotFoundException.class })
    public String handler() {
        throw new RuntimeException("运行时异常。。。。");
    }

    public String handlerFailback(Throwable e) {
        System.err.println("异常信息：" + e.getMessage());
        return "获取异常信息并可以做具体的降级处理";
    }

    @HystrixCommand(fallbackMethod = "callRequestFailback")
    public String callRequest() {
        return restTemplate.getForEntity("http://client/request", String.class).getBody();
    }

    public String callRequestFailback() {
        System.err.println("callRequest 执行降级策略");
        return "callRequestFailback";
    }

}

```
```Java
@RestController
public class TestController {

    @Autowired
    private HelloService helloService;

    @RequestMapping(value = "/hystrix-hello", method = RequestMethod.GET)
    public String hello() {
        return helloService.callHello();
    }

    @RequestMapping(value = "/hystrix-handler", method = RequestMethod.GET)
    public String handler() {
        return helloService.handler();
    }

    @RequestMapping(value = "/hystrix-request", method = RequestMethod.GET)
    public String request() {
        return helloService.callRequest();
    }
}

```

在 `application.yml` 中配置 Hystrix 的超时时间。
```YML
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 2000 # 默认值为 1 秒
```
{% note warning %}
使用包装 Ribbon 客户端的 Hystrix 命令时，要确保将 Hystrix 超时配置为比配置的 Ribbon 超时（包括可能进行的任何潜在重试）更长。 例如，如果您的 Ribbon 连接超时时间为一秒，并且 Ribbon 客户端可能会重试三次请求，那么您的 Hystrix 超时应略超过三秒钟。
{% endnote %}

http://localhost:9001/hystrix-hello 返回 `callHelloFailback`

http://localhost:9001/hystrix-handler 返回 `获取异常信息并可以做具体的降级处理`

http://localhost:9001/hystrix-request 返回 `request`

除了全局配置 Hystrix 参数，我们也可以根据具体的方法配置不同的参数。在通过注解 `@HystrixCommand` 中的  `commandKey` 与 `commandProperties` 属性相互结合使用，可以单独的控制特殊需求的实现。比如我这里设置超时时间为 8 秒，在 `HelloService` 添加方法：
```Java
@HystrixCommand(commandKey = "hiKey", commandProperties = {
        @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "8000") },
         fallbackMethod = "callHiFailback")
public String callHi(String name) {
    return restTemplate.getForEntity("http://client/hi?name=" + name, String.class).getBody();
}

public String callHiFailback(String name) {
    System.err.println("callHi 执行降级策略");
    return "callHiFailback";
}
```
在 `TestController` 添加：
```Java
@RequestMapping(value = "/hystrix-hi", method = RequestMethod.GET)
public String hi(String name) {
    return helloService.callHi(name);
}

```

访问 http://localhost:9001/hystrix-hi?name=tom 返回 `hi tom!`

更多的配置可以去 [Hystrix wiki](https://github.com/Netflix/Hystrix/wiki/Configuration) 查看。

## 断路器
> 下面几段均摘抄于 [Spring Cloud构建微服务架构：服务容错保护（Hystrix断路器）【Dalston版】](http://blog.didispace.com/spring-cloud-starter-dalston-4-3/) 有小幅改动

在服务消费端的服务降级逻辑因为 Hystrix 命令调用依赖服务超时，触发了降级逻辑，但是即使这样，受限于 Hystrix 超时时间的问题，我们的调用依然很有可能产生堆积。

这个时候断路器就会发挥作用，那么断路器是在什么情况下开始起作用呢？这里涉及到断路器的四个重要参数：
* 快照时间窗（`metrics.rollingStats.timeInMilliseconds`）：断路器确定是否打开需要统计一些请求和错误数据，而统计的时间范围就是快照时间窗，默认为最近的 10 秒
* 请求总数下限（`circuitBreaker.requestVolumeThreshold`）：在快照时间窗内，必须满足请求总数下限才有资格根据熔断。默认为20，意味着在 10 秒内，如果该 Hystrix 命令的调用此时不足20次，即时所有的请求都超时或其他原因失败，断路器都不会打开
* 错误百分比下限（`circuitBreaker.errorThresholdPercentage`）：当请求总数在快照时间窗内超过了下限，比如发生了 30 次调用，如果在这 30 次调用中，有 16 次发生了超时异常，也就是超过 50% 的错误百分比，在默认设定 50% 下限情况下，这时候就会将断路器打开
* 休眠时间窗（`circuitBreaker.sleepWindowInMilliseconds`）：也可以理解为短路时间，默认为 5 秒。

那么当断路器打开之后会发生什么呢？我们先来说说断路器未打开之前，对于之前那个示例的情况就是每个请求都会在当 Hystrix 超时之后返回 fallback ，每个请求时间延迟就是近似 Hystrix 的超时时间，如果设置为 5 秒，那么每个请求就都要延迟 5 秒才会返回。当熔断器在 10 秒内发现请求总数超过 20，并且错误百分比超过 50%，这个时候熔断器打开。打开之后，再有请求调用的时候，将不会调用主逻辑，而是直接调用降级逻辑，这个时候就不会等待 5 秒之后才返回 fallback 。通过断路器，实现了自动地发现错误并将降级逻辑切换为主逻辑，减少响应延迟的效果。

在断路器打开之后，处理逻辑并没有结束，我们的降级逻辑已经被成了主逻辑，那么原来的主逻辑要如何恢复呢？对于这一问题，Hystrix 也为我们实现了自动恢复功能。当断路器打开，对主逻辑进行熔断之后，Hystrix 会启动一个休眠时间窗，在这个时间窗内，降级逻辑是临时的成为主逻辑，当休眠时间窗到期，断路器将进入半开状态，释放一次请求到原来的主逻辑上，如果此次请求正常返回，那么断路器将继续闭合，主逻辑恢复，如果这次请求依然有问题，断路器继续进入打开状态，休眠时间窗重新计时。

通过上面的一系列机制，Hystrix 的断路器实现了对依赖资源故障的端口、对降级策略的自动切换以及对主逻辑的自动恢复机制。这使得我们的微服务在依赖外部服务或资源的时候得到了非常好的保护，同时对于一些具备降级逻辑的业务需求可以实现自动化的切换与恢复，相比于设置开关由监控和运维来进行切换的传统实现方式显得更为智能和高效。

写个方法测试下：

```Java
    @HystrixCommand(fallbackMethod = "callRequestFailback",
            commandKey="requestKey",
            commandProperties= {
                    @HystrixProperty(name="circuitBreaker.requestVolumeThreshold",value="10"),
                    @HystrixProperty(name="circuitBreaker.errorThresholdPercentage",value="20"),
                    @HystrixProperty(name="circuitBreaker.sleepWindowInMilliseconds",value="5000")
                    
            })
    public String callRequest() {
        System.out.println("-----非降级-----");
        return restTemplate.getForEntity("http://client/request", String.class).getBody();
    }

```
```Java
public static void main(String[] args) {
       RestTemplate rt = new RestTemplate();
       CountDownLatch cd = new CountDownLatch(10);
       for(int i=0;i<10;i++) {
           new Thread(()-> {
               int random = new Random().nextInt(3);
               try {
                TimeUnit.SECONDS.sleep(random);
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
               cd.countDown();
               System.out.println(cd.getCount()+": "+rt.getForEntity("http://localhost:9001/hystrix-request", String.class).getBody());
             
           }).start();
       }
       
       try {
        cd.await();
    } catch (InterruptedException e) {
        // TODO Auto-generated catch block
        e.printStackTrace();
    }
       long start = System.currentTimeMillis();
       System.out.println(rt.getForEntity("http://localhost:9001/hystrix-request", String.class).getBody());
       long end = System.currentTimeMillis();
       System.out.println("*******************"+(end - start));
       
    }
```


## 更多功能

由于断路器功能太过强大，这里仅仅举一个例子说明：高并发限流系统/限流组件应用，在断路器的配置限流策略 `execution.isolation.stargery` 中有两种方式进行处理：
* thread：通过线程池隔离的策略，它会独立在一个线程上执行，并且它的并发量受线程池数量的限制
* semaphone：它则实现在调用的线程上，通过信号量的方式进行隔离，这种则类似 java 的限流，受到信号量计数器的限制所约束

线程的策略方式：
```Java
@HystrixCommand(commandKey = "threadKey",
         commandProperties = {
            @HystrixProperty(name = "execution.isolation.strategy",value = "THREAD") ,
            @HystrixProperty(name = "coreSize",value = "10"),
            @HystrixProperty(name = "maxQueueSize",value = "50"),//最大阈值
            @HystrixProperty(name = "queueSizeRejectionThreshold",value = "30")//拒绝阈值
            },
            fallbackMethod = "callHiFailback")
```

信号量的策略方式：
```Java
@HystrixCommand(commandKey = "semaphoreKey",
         commandProperties = {
            @HystrixProperty(name = "execution.isolation.strategy",value = "SEMAPHORE") ,
            @HystrixProperty(name = "execution.isolation.semaphore.maxConcurrentRequests",value = "50")//拒绝阈值
            },
            fallbackMethod = "callHiFailback")
```

## Dashboard

## Turbine




<!-- 额外赠送 Spring Retry 断路器 -->

## 示例源码

本篇文章所用示例项目名称均以 `spring-cloud-03-hystrix` 开头

GitHub：https://github.com/yangdd1205/spring-cloud-master

## 参考资料



