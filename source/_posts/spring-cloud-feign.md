---
title: Spring Cloud Feign：声明式 REST 客户端
tags:
  - Spring Cloud
categories: Spring Cloud
date: 2017-12-31 17:02:41
---


前面我们讲了使用 RestTemplate + Ribbon + Hystrix 进行服务的调用，其实还有更简单的方法，通过 Feign 调用。
<!-- more -->

## Feign

Feign 是一个声明式的 WEB 服务客户端，它让编写 WEB 服务客户端更加简单。只需要创建一个接口并使用注解进行配置即可。它支持插入式注解，包含了 Feign 注解和 JAX-RS 注解。Feign 还支持可插拔编码器和解码器。而 Spring Cloud Feing 增加了对 Spring MVC 注解的支持，并使用了 Spring Web 中默认使用的 `HttpMessageConverters`，还整合了 Ribbon 和 Eureka 来提供一个具有负载均衡的 HTTP 客户端。

## 使用

现有四个项目：
* [server](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-04-feign-consumer)：注册中心，服务名：eureka-service，端口：7001
* [provider-a](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-04-feign-provider-a)：服务提供者，服务名：provider，端口：8001
* [provider-b](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-04-feign-provider-b)：服务提供者，服务名：provider，端口：8002
* [consumer](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-04-feign-consumer)：服务消费者，服务名：consumer，端口：9001

`provider` 提供的服务：
```Java
@RestController
public class HelloController {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String sayHello() {
        System.out.println("provider-a hello feign!");
        return "hello feign!";
    }
    
    @RequestMapping(value = "/hi", method = RequestMethod.GET)
    public String sayHi() throws InterruptedException {
        TimeUnit.SECONDS.sleep(5);
        System.out.println("provider-a hi feign!");
        return "hi feign!";
    }
}
```
注意：`provider-b` 中需要打印 `provider-b`。

在 `consumer` 中，我们首先引入 Feign 依赖：
```XML
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
然后再主入口通过注解 `@EnableFeignClients` 启用 Feign Client 功能。
```Java
@EnableFeignClients(basePackages={"org.yangdongdong.springcloud.*"}) // 开启 Feign client 功能
@EnableDiscoveryClient
@SpringBootApplication
public class Application {

    /**
     * 注意：使用 feign 的时候，不需要去做自定义的 RestTemplate 配置。
     * 
     * @return
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

Spring Cloud Feign 还集成了 Hystrix，不过默认是禁用的，在源码 `HystrixFeignConfiguration` 类中可以找到，我们要在配置文件中手动开启：
```YML
feign:
  hystrix:
    enabled: true # 启动 feign 的断路器功能
  compression:
    request:
      mime-types: # 支持的格式
      - text/html,application/xml,application/json
      min-request-size: 2048 # 超过就压缩
      enabled: true # 请求是否压缩
    response:
      enabled: true # 响应是否压缩
```
现在我们通过 Feign 调用 `provider` 提供的服务，通过注解 `@FeignClient` 表明是一个 Feign Client，配置注解 `name` 指明服务的提供者，`fallback` 表示服务降级方法所在类。
```Java
@FeignClient(name = "provider", fallback = ProviderFeignClientHystrixFallback.class)
public interface ProviderFeignClient {
    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String sayHello();

    @RequestMapping(value = "/hi", method = RequestMethod.GET)
    public String sayHi();
}
```
```Java
@Component // 不要忘记 @Component 注解
public class ProviderFeignClientHystrixFallback implements ProviderFeignClient {

    @Override
    public String sayHello() {
        return "Hello Fallback";
    }

    @Override
    public String sayHi() {
        return "Hi Fallback";
    }

}
```
```Java
@RestController
public class ConsumerController {
    
    @Autowired
    private ProviderFeignClient helloFeignClient;
    
    @RequestMapping(value="/hello",method=RequestMethod.GET)
    public String hello() {
        return helloFeignClient.sayHello();
    }
    
    @RequestMapping(value="/hi",method=RequestMethod.GET)
    public String hi() {
        return helloFeignClient.sayHi();
    }
}
```

依次启动项目，访问：http://localhost:9001/hello 观察 `provider-a` 和 `provider-b` 的控制台，发现是通过轮询来调用的服务，证实了 Fegin 集成了 Ribbon 具有负载均衡的功能。

再访问：http://localhost:9001/hi 返回 `Hi Fallback`，但是发现 `provider-a` 和 `provider-b` 的控制台都有打印调用信息。原因是，Feign 底层默认提供了重试机制，也就是底层使用 Retry 类对调用服务进行重试操作，通过底层代码 `Retryer.java` 我们知道默认是 1000ms 去进行调用，调用次数是 5 次。我们可以配置 Ribbon 和 Hystrix 的超时时间。

{% note warning %}
在配置 Ribbon 和 Hystrix 超时时间时，要特别注意，要确保将 Hystrix 超时配置为比配置的 Ribbon 超时（包括可能进行的任何潜在重试）更长。 例如，如果您的 Ribbon 连接超时时间为一秒，并且 Ribbon 客户端可能会重试三次请求，那么您的 Hystrix 超时应略超过三秒钟。如果 Hystrix 的超时时间小于 Ribbon 超时时间，则不会重试，直接被断路器组件调用请求执行熔断，服务降级。
{% endnote %}

```YML
## 断路器超时时间
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 30000
provider:
  ribbon:
    OkToRetryOnAllOperations: true # 允许所有该服务的所有操作都可以重试 
    MaxAutoRetriesNextServer: 1  # 参与重试的服务个数（不包含第一个服务）
    MaxAutoRetries: 5 # 重试次数（不包含第一次请求）
    # 这两个属性在单独使用 Ribbon 时，是不生效的，但是使用 Feign 就行
    ConnectTimeout: 2000   # 连接超时时间
    ReadTimeout: 3000     # 处理超时时间


##### 发现深坑 /(ㄒoㄒ)/~~ ，一定要加下面这段配置 ######

# disable Ribbon's cicruit breaker and rely soley on Hystrix.
# this helps to avoid confusion.
# see https://github.com/Netflix/ribbon/issues/15
niws:
  loadbalancer:
    availabilityFilteringRule:
      filterCircuitTripped: false # 默认为 true
```

访问：http://localhost:9001/hi 查看 `provider-a` 和 `provider-b` 的控制台都有打印调用信息。

## 踩坑

如果我们把 `filterCircuitTripped` 注释掉或改为 `true`。重启服务，访问：http://localhost:9001/hi `consumer` 会报如下的错误：
```Java
2017-12-31 16:12:55.410  WARN 12948 --- [trix-provider-1] c.netflix.loadbalancer.BaseLoadBalancer  : LoadBalancer [provider]:  Error choosing server for key null

java.lang.IndexOutOfBoundsException: index (1) must be less than size (1)
    at com.google.common.base.Preconditions.checkElementIndex(Preconditions.java:310) ~[guava-18.0.jar:na]
    at com.google.common.base.Preconditions.checkElementIndex(Preconditions.java:292) ~[guava-18.0.jar:na]
    at com.google.common.collect.SingletonImmutableList.get(SingletonImmutableList.java:45) ~[guava-18.0.jar:na]
    at com.netflix.loadbalancer.AbstractServerPredicate.chooseRoundRobinAfterFiltering(AbstractServerPredicate.java:203) ~[ribbon-loadbalancer-2.2.4.jar:2.2.4]
    at com.netflix.loadbalancer.PredicateBasedRule.choose(PredicateBasedRule.java:45) ~[ribbon-loadbalancer-2.2.4.jar:2.2.4]
    at com.netflix.loadbalancer.BaseLoadBalancer.chooseServer(BaseLoadBalancer.java:736) ~[ribbon-loadbalancer-2.2.4.jar:2.2.4]
    at com.netflix.loadbalancer.ZoneAwareLoadBalancer.chooseServer(ZoneAwareLoadBalancer.java:113) [ribbon-loadbalancer-2.2.4.jar:2.2.4]
    at com.netflix.loadbalancer.LoadBalancerContext.getServerFromLoadBalancer(LoadBalancerContext.java:481) [ribbon-loadbalancer-2.2.4.jar:2.2.4]
    at com.netflix.loadbalancer.reactive.LoadBalancerCommand$1.call(LoadBalancerCommand.java:184) [ribbon-loadbalancer-2.2.4.jar:2.2.4]
    at com.netflix.loadbalancer.reactive.LoadBalancerCommand$1.call(LoadBalancerCommand.java:180) [ribbon-loadbalancer-2.2.4.jar:2.2.4]
    at rx.Observable.unsafeSubscribe(Observable.java:10151) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeConcatMap.call(OnSubscribeConcatMap.java:94) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeConcatMap.call(OnSubscribeConcatMap.java:42) [rxjava-1.2.0.jar:1.2.0]
    at rx.Observable.unsafeSubscribe(Observable.java:10151) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OperatorRetryWithPredicate$SourceSubscriber$1.call(OperatorRetryWithPredicate.java:127) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.schedulers.TrampolineScheduler$InnerCurrentThreadScheduler.enqueue(TrampolineScheduler.java:73) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.schedulers.TrampolineScheduler$InnerCurrentThreadScheduler.schedule(TrampolineScheduler.java:52) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OperatorRetryWithPredicate$SourceSubscriber.onNext(OperatorRetryWithPredicate.java:79) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OperatorRetryWithPredicate$SourceSubscriber.onNext(OperatorRetryWithPredicate.java:45) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.util.ScalarSynchronousObservable$WeakSingleProducer.request(ScalarSynchronousObservable.java:276) [rxjava-1.2.0.jar:1.2.0]
    at rx.Subscriber.setProducer(Subscriber.java:209) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.util.ScalarSynchronousObservable$JustOnSubscribe.call(ScalarSynchronousObservable.java:138) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.util.ScalarSynchronousObservable$JustOnSubscribe.call(ScalarSynchronousObservable.java:129) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30) [rxjava-1.2.0.jar:1.2.0]
    at rx.Observable.subscribe(Observable.java:10247) [rxjava-1.2.0.jar:1.2.0]
    at rx.Observable.subscribe(Observable.java:10214) [rxjava-1.2.0.jar:1.2.0]
    at rx.observables.BlockingObservable.blockForSingle(BlockingObservable.java:444) [rxjava-1.2.0.jar:1.2.0]
    at rx.observables.BlockingObservable.single(BlockingObservable.java:341) [rxjava-1.2.0.jar:1.2.0]
    at com.netflix.client.AbstractLoadBalancerAwareClient.executeWithLoadBalancer(AbstractLoadBalancerAwareClient.java:112) [ribbon-loadbalancer-2.2.4.jar:2.2.4]
    at org.springframework.cloud.netflix.feign.ribbon.LoadBalancerFeignClient.execute(LoadBalancerFeignClient.java:63) [spring-cloud-netflix-core-1.4.0.RELEASE.jar:1.4.0.RELEASE]
    at feign.SynchronousMethodHandler.executeAndDecode(SynchronousMethodHandler.java:97) [feign-core-9.5.0.jar:na]
    at feign.SynchronousMethodHandler.invoke(SynchronousMethodHandler.java:76) [feign-core-9.5.0.jar:na]
    at feign.hystrix.HystrixInvocationHandler$1.run(HystrixInvocationHandler.java:108) [feign-hystrix-9.5.0.jar:na]
    at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:302) [hystrix-core-1.5.12.jar:1.5.12]
    at com.netflix.hystrix.HystrixCommand$2.call(HystrixCommand.java:298) [hystrix-core-1.5.12.jar:1.5.12]
    at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:46) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:35) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30) [rxjava-1.2.0.jar:1.2.0]
    at rx.Observable.unsafeSubscribe(Observable.java:10151) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:51) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeDefer.call(OnSubscribeDefer.java:35) [rxjava-1.2.0.jar:1.2.0]
    at rx.Observable.unsafeSubscribe(Observable.java:10151) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeDoOnEach.call(OnSubscribeDoOnEach.java:41) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeDoOnEach.call(OnSubscribeDoOnEach.java:30) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:48) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OnSubscribeLift.call(OnSubscribeLift.java:30) [rxjava-1.2.0.jar:1.2.0]
    at rx.Observable.unsafeSubscribe(Observable.java:10151) [rxjava-1.2.0.jar:1.2.0]
    at rx.internal.operators.OperatorSubscribeOn$1.call(OperatorSubscribeOn.java:94) [rxjava-1.2.0.jar:1.2.0]
    at com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction$1.call(HystrixContexSchedulerAction.java:56) [hystrix-core-1.5.12.jar:1.5.12]
    at com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction$1.call(HystrixContexSchedulerAction.java:47) [hystrix-core-1.5.12.jar:1.5.12]
    at com.netflix.hystrix.strategy.concurrency.HystrixContexSchedulerAction.call(HystrixContexSchedulerAction.java:69) [hystrix-core-1.5.12.jar:1.5.12]
    at rx.internal.schedulers.ScheduledAction.run(ScheduledAction.java:55) [rxjava-1.2.0.jar:1.2.0]
    at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [na:1.8.0_131]
    at java.util.concurrent.FutureTask.run(FutureTask.java:266) [na:1.8.0_131]
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142) [na:1.8.0_131]
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617) [na:1.8.0_131]
    at java.lang.Thread.run(Thread.java:748) [na:1.8.0_131]
```

根据错误信息，一层一层的找，我们首先看
{% codeblock AbstractServerPredicate.java lang:java%}
public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
    List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
    if (eligible.size() == 0) {
        return Optional.absent();
    }
    return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
}
{% endcodeblock %}

在我们第一次访问是 `eligible` 的大小是 2，但进行重试的时候，大小就变成了 1。由于 Ribbon 是采用轮询方法做的负载均衡，具体的代码可以看 `incrementAndGetModulo(eligible.size())`，所以第二次返回的是 1，让去取 eligible 下标为 1 的元素。（黑人问号.jpg）当前 `eligible` 的大小是 1，所有报错了下标越界。

那为啥两次返回的 Server 列表个数不一样呢？明明我们 `provider` 有两个实例。再找看代码：
{% codeblock AbstractServerPredicate.java lang:java%}
public List<Server> getEligibleServers(List<Server> servers, Object loadBalancerKey) {
    if (loadBalancerKey == null) {
        // 发现这里对服务列表做了过滤  
        return ImmutableList.copyOf(Iterables.filter(servers, thi.getServerOnlyPredicate()));           
    } else {
        List<Server> results = Lists.newArrayList();
        for (Server server: servers) {
            if (this.apply(new PredicateKey(loadBalancerKey, server))) {
                results.add(server);
            }
        }
        return results;            
    }
}
{% endcodeblock %}

继续一层一层的找，直到找到发现了过滤的代码：
{% codeblock AbstractServerPredicate.java lang:java%}
@Override
public boolean apply(@Nullable PredicateKey input) {
    LoadBalancerStats stats = getLBStats();
    if (stats == null) {
        return true;
    }
    return !shouldSkipServer(stats.getSingleServerStat(input.getServer()));
}


private boolean shouldSkipServer(ServerStats stats) {        
    if ((CIRCUIT_BREAKER_FILTERING.get() && stats.isCircuitBreakerTripped()) 
            || stats.getActiveRequestsCount() >= activeConnectionsLimit.get()) {
        return true;
    }
    return false;
}
{% endcodeblock %}

主要看 `shouldSkipServer` 方法中的 `CIRCUIT_BREAKER_FILTERING.get()` 断路器过滤？？进去看看：
{% codeblock AvailabilityPredicate.java lang:java%}
/**
* Choose a server in a round robin fashion after the predicate filters a given list of servers and load balancer key. 
*/
public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
    private static final DynamicBooleanProperty CIRCUIT_BREAKER_FILTERING =
            DynamicPropertyFactory.getInstance().
            // 过滤断路器打开服务列表
            getBooleanProperty("niws.loadbalancer.availabilityFilteringRule.filterCircuitTripped", true);
}
{% endcodeblock %}

终于找到原因了，喜（F）极（U）而（C）泣（K）！


好了，这应该是 2017 年最后一篇博客。

**再见 2107，你好 2018**

## 示例源码

本篇文章所用示例项目名称均以 `spring-cloud-04-feign` 开头

GitHub：https://github.com/yangdd1205/spring-cloud-master

## 参考资料
[Feign](https://github.com/OpenFeign/feign)
[Spring Cloud Feign](http://cloud.spring.io/spring-cloud-static/Edgware.RELEASE/single/spring-cloud.html#spring-cloud-feign)
