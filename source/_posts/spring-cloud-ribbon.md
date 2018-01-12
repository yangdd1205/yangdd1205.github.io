---
title: Spring Cloud Ribbon：负载均衡
tags:
  - Spring Cloud
categories: Spring Cloud
date: 2017-12-23 18:22:24
---


负载均衡对于一个系统架构来说非常重要，是必须要有的一个基础设施，它能够有效的缓解网络压力和流量扩容。我们知道的负载均衡可以分为两种，一种是硬件负载均衡，比如 F5 服务器或者 SLB 负载设施；另外一种是软件负载均衡设施，比如我们耳熟能详的 Nginx 反向代理、LVS 负载均衡、HAProxy 负载均衡等等。但是只要是负载均衡服务都是能以类似的架构方式来构建
![负载均衡架构](http://7xq43s.com1.z0.glb.clouddn.com/LoadBalanced.png "负载均衡架构")
<!-- more -->

## Ribbon

Ribbon 是一个客户端负载均衡器，它让你对 HTTP 和 TCP 客户端的行为有更大的控制权。Feign 已经使用了 Ribhon 做负载均衡。当 Ribbon 与 Eureka 一起使用时，`ribbonServerList` 会被 `DiscoveryEnabledNIWSServerList` 重写，扩展成从 Eureka 注册中心获取服务实例列表。同时它也会用 `NIWSDiscoveryPing` 来取代 `IPing`，它将职责委托给 Eureka 来确定服务端是否已经启动，默认使用轮询访问的规则来达到负载均衡的作用。关于负载均衡可以看看这篇文章 [ Working with load balancers](https://github.com/Netflix/ribbon/wiki/Working-with-load-balancers)。

## RestTemplate

要学习 Ribbon ，我们必须先了解 `RestTemplate` 的使用，因为 Ribbon 是基于  `RestTemplate` 模板进行负载调用的。其实跟 HttpClient 差不多，只不过在 `RestTemplate` 中，基本上都对 GET、PUT、POST、DELETE 进行了重载，提供了更多更简单的调用接口。

### 简单使用
基于上一篇文章的示例项目，我们创建以下 6 个项目：
* 注册中心：eureka-a、eureka-b
* 服务提供者：clien-1、client-2
* 服务调用者：request、retry

具体的请到 [GitHub](https://github.com/yangdd1205/spring-cloud-master) 中找以 `spring-cloud-02-ribbon` 开头的项目。 

在 `clinet-1` 项目中创建
```Java
package org.yangdongdong.springcloud.service.api;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class IndexController {

  @RequestMapping(value = "/sayHello", method = RequestMethod.GET)
  public String sayHello(String name) {
    return "hello " + name;
  }

}
```

依次启动 `eureka-a`、`eureka-b`、`client-1`，访问 http://localhost:8001/sayHello?name=Tom 测试下。

我们试试使用 `RestTemplate` 怎么调用 `sayHello`：
```Java
public static void main(String[] args) {
    RestTemplate restTemplate = new RestTemplate();
    
    ResponseEntity<String> response = restTemplate.getForEntity("http://localhost:8001/sayHello?name=Tom",String.class);

    System.out.println(response.getBody());
    
    Map<String, String> map = new HashMap<String, String>();
    map.put("name","Jack");
    String result = restTemplate.getForObject("http://localhost:8001/sayHello?name={name}", String.class, map);
    System.out.println(result);
}
```
输出
```
17:40:51.957 [main] DEBUG org.springframework.web.client.RestTemplate - Created GET request for "http://localhost:8001/sayHello?name=Tom"
17:40:51.967 [main] DEBUG org.springframework.web.client.RestTemplate - Setting request Accept header to [text/plain, application/json, application/*+json, */*]
17:40:52.139 [main] DEBUG org.springframework.web.client.RestTemplate - GET request for "http://localhost:8001/sayHello?name=Tom" resulted in 200 (null)
17:40:52.140 [main] DEBUG org.springframework.web.client.RestTemplate - Reading [java.lang.String] as "text/plain;charset=UTF-8" using [org.springframework.http.converter.StringHttpMessageConverter@78ac1102]
hello Tom
17:40:52.144 [main] DEBUG org.springframework.web.client.RestTemplate - Created GET request for "http://localhost:8001/sayHello?name=Jack"
17:40:52.144 [main] DEBUG org.springframework.web.client.RestTemplate - Setting request Accept header to [text/plain, application/json, application/*+json, */*]
17:40:52.243 [main] DEBUG org.springframework.web.client.RestTemplate - GET request for "http://localhost:8001/sayHello?name=Jack" resulted in 200 (null)
17:40:52.243 [main] DEBUG org.springframework.web.client.RestTemplate - Reading [java.lang.String] as "text/plain;charset=UTF-8" using [org.springframework.http.converter.StringHttpMessageConverter@78ac1102]
hello Jack

```

### 负载均衡

要创建一个负载均衡的 `RestTemplate`，使用 `@Bean` 并使用 `@LoadBalanced` 限定符。

首先在 `request` 项目加入 Ribbon 的依赖。
```XML
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

修改 `RequestApplication`：

```Java
@EnableDiscoveryClient
@SpringBootApplication
public class RequestApplication {

    @Bean
    @LoadBalanced   //自动负载均衡：他的机制是（通过）Application Name去寻找服务发现，然后去做负载均衡策略的
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    public static void main(String[] args) {
        SpringApplication.run(RequestApplication.class, args);
    }
}
```

创建调用服务的方法：
```Java
package org.yangdongdong.springcloud.service.api;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

public class TestController {

    @Autowired
    private RestTemplate restTemplate;

    /**
     * 调用 Spring Cloud 服务
     * 
     * @param name
     * @return
     */
    @RequestMapping(value = "callByServerUrl", method = RequestMethod.GET)
    public String callByServerUrl(String name) {
        // 调用地址：http://[请求协议头] client [服务名] / [context-path] /sayHello [调用方法的地址]
        ResponseEntity<String> response = restTemplate.getForEntity("http://client/sayHello?name=" + name,
                String.class);
        return response.getBody();
    }

    /**
     * 采用 RestTemplate 方式调用（HTTP方式调用，与 Spring Cloud 调用服务没有一点关系）
     * 
     * @param name
     * @return
     */
    @RequestMapping(value = "callByHttpUrl", method = RequestMethod.GET)
    public String callByHttpUrl(String name) {
        RestTemplate restTemplate = new RestTemplate();
        ResponseEntity<String> response = restTemplate.getForEntity("http://localhost:8001/sayHello?name=" + name,
                String.class);
        return response.getBody();
    }
}
```

访问相应链接测试。

{% note info %}
 可以试试将 `callByHttpUrl` 方法中的 `restTemplate` 换成自动注入的 `restTemplate` 看看有什么效果。
{% endnote %}

现在 `client` 服务只有一个，无法测试负载均衡，所以我们把 `client-1` 的 `IndexController` 复制到 `client-2` 并加入相应的标识用于区分调用的是哪个服务。

现在访问 http://localhost:9001/callByServerUrl?name=Tom 10 次。

`client-1` 的控制台输出
```Java
client 1
client 1
client 1
client 1
client 1
```

`client-2` 的控制台输出
```Java
client 2
client 2
client 2
client 2
client 2
```

{% note info %}
 现在除了测试负载均衡，也可以测试上篇文章所说的高可用。
{% endnote %}


## 失败重试

对于互联网的服务调用，通常情况下我们都需要对某些服务进行重试策略，比如重试多次失败则返回错误信息，当然可重试的服务接口必须要自己实现**幂等**，不然由于网络或者其他原因导致请求时间过长导致重复操作，会出现我们不想要的结果。

在 Spring 中有个很经典的框架就是 Retry，使用非常简单，只需要加上 `@EnableRetry` 启用重试机制，并且 `@Retryable` 注解配置重试所需要的服务即可，可以自己定义重试的异常信息、重试次数以及重试策略执行的周期，非常的强大好用。并且在最终重试次数都完成调用且仍然调用服务失败时，我们可以使用 `@Recover` 注解来完成最终的记录和补偿处理。


{% note warning %}
重试策略会有单点问题，我们必须要配合持久化的相关标识来判断单点问题的重试策略算法搞定了，不然我们没有达到使用 Spring Retry 的效果。当然了，我们基于 Ribbon 的重试机制则不会存在该问题，即使存在也不需要考虑，因为我们仅仅是 http 服务调用而已，并不是持久化操作或涉及其他相关核心操作。
{% endnote %}

在 `retry` 项目中加入依赖：

```XML
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
</dependency>
```

在 `RetryApplication` 加上 `@EnableRetry` 注解。

我们简单地使用下：
```Java
package org.yangdongdong.springcloud.service;

import org.springframework.remoting.RemoteAccessException;
import org.springframework.retry.annotation.Backoff;
import org.springframework.retry.annotation.Recover;
import org.springframework.retry.annotation.Retryable;
import org.springframework.stereotype.Service;

@Service
public class RetryService {

    // 方法：需要进行重试[远程调动服务失败的情况，再次进行调用]
    @Retryable(value = { RemoteAccessException.class }, // value：什么样的异常进行重试策略的执行
            maxAttempts = 4, // maxAttempts：重试次数 包含首次调用失败 默认值：3
            backoff = @Backoff(delay = 2000, multiplier = 2)) // delay：间隔 multiplier：递增倍数（即下次间隔是上次的多少倍）
    public void call() throws Exception {
        System.err.println("时间：" + (System.currentTimeMillis() / 1000) + " 调用了 call 方法执行。。。。");
        throw new RemoteAccessException("RPC 调用异常");
    }

    @Recover
    public void revocer(RemoteAccessException e) {
        System.err.println("------ 最终处理 ------：" + e.getMessage());
    }
}

```

测试下：
```Java
package org.yangdongdong.springcloud.service;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.yangdongdong.springcloud.RetryApplication;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = RetryApplication.class)
public class RetryServiceTest {

    @Autowired
    private RetryService retryservice;

    @Test
    public void testCall() throws Exception{
        retryservice.call();
    }

}

```

输出：
```
时间：1514019200 调用了 call 方法执行。。。。
时间：1514019202 调用了 call 方法执行。。。。
时间：1514019206 调用了 call 方法执行。。。。
时间：1514019214 调用了 call 方法执行。。。。
------ 最终处理 ------：RPC 调用异常
```


### 与 Ribbon 整合

我们再看看与 Ribbon 整合。

在 `client-1` 项目的 `IndexController` 中添加如下方法：
```Java
@RequestMapping(value = "/retry", method = RequestMethod.GET)
public String retry() {
    System.err.println("client 1 call.......");
    try {
        // 模拟连接超时
        TimeUnit.SECONDS.sleep(6);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "client 1";
}
```
而在 `client-2`项目中的 `IndexController` 中则正常返回：
```Java
@RequestMapping(value = "/retry", method = RequestMethod.GET)
public String retry() {
    System.err.println("client 2 call.......");
    return "client 2";
}
```

在 `retry` 项目中调用服务：
```Java
package org.yangdongdong.springcloud.service.api;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

@RestController
public class RetryController {

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping(value = "retry", method = RequestMethod.GET)
    public String retry() {
        ResponseEntity<String> response = restTemplate.getForEntity("http://client/retry", String.class);
        String result = response.getBody();
        System.err.println("result: " + result);
        return "result: " + result;
    }
}
```

在 `retry` 项目中配置服务的重试参数：
```YML
# 格式 <clientName>.<nameSpace>.<propertyName>=<value>
client: # 服务名
  ribbon:
    OkToRetryOnAllOperations: true # 允许所有该服务的所有操作都可以重试 
    MaxAutoRetriesNextServer: 1  # 参与重试的服务个数（不包含第一个服务）
    MaxAutoRetries: 5 # 重试次数（不包含第一次请求）
```

配置超时时间，有两种方式，一种是 Java 代码，一种是配置文件。

方式一：
```Java
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory httpRequestFactory = new HttpComponentsClientHttpRequestFactory();
    httpRequestFactory.setConnectionRequestTimeout(1000);
    httpRequestFactory.setConnectTimeout(1000);
    httpRequestFactory.setReadTimeout(3000);
    return new RestTemplate(httpRequestFactory);
}
```

方式二：

在配置文件中添加：
```YML
custom:
  http: 
    connect-timeout: 1000
    connection-request-timeout: 1000 
    read-timeout: 3000 
```
```Java
@Bean
@ConfigurationProperties(prefix = "custom.http")
public HttpComponentsClientHttpRequestFactory httpRequestFactory() {
    return new HttpComponentsClientHttpRequestFactory();
}

@Bean
@LoadBalanced // 自动负载均衡：他的机制是（通过）Application Name去寻找服务发现，然后去做负载均衡策略的
public RestTemplate restTemplate() {
    return new RestTemplate(httpRequestFactory());
}
```

一切准备就绪，启动项目。访问：http://localhost:9002/retry

查看控制台（下面的输出基于第一次就调用的是 `client-1`）

`client-1` 控制台输出：
```
client 1 call.......
client 1 call.......
client 1 call.......
client 1 call.......
client 1 call.......
client 1 call.......
```
可以看到，第一次打印是正常调用，后面的 5 次是失败重试。

`client-2` 控制台输出：
```
client 2 call.......
```
当 5 次重试还是失败，切换到了 `client-2`

`retry` 控制台输出：
```
result: client 2
```

控制台只会打印 `result: client 2`。由此可见失败重试生效了。


{% note info %}
 假如，把 `client-2` 也休眠 6 秒种，会出现什么情况呢？
{% endnote %}



## 示例源码

本篇文章所用示例项目名称均以 `spring-cloud-02-ribbon` 开头

GitHub：https://github.com/yangdd1205/spring-cloud-master

## 参考资料
[Spring Cloud构建微服务架构：服务消费（Ribbon）【Dalston版】](http://blog.didispace.com/spring-cloud-starter-dalston-2-2)
[为Spring Cloud Ribbon配置请求重试（Camden.SR2+）](http://blog.didispace.com/spring-cloud-ribbon-failed-retry/)
[Spring Cloud Commons: Common Abstractions](http://cloud.spring.io/spring-cloud-static/Edgware.RELEASE/single/spring-cloud.html#_spring_cloud_commons_common_abstractions)
[Client Side Load Balancer: Ribbon](http://cloud.spring.io/spring-cloud-static/Edgware.RELEASE/single/spring-cloud.html#spring-cloud-ribbon)
[Ribbon documentation](https://github.com/Netflix/ribbon/wiki/Getting-Started#the-properties-file-sample-clientproperties)