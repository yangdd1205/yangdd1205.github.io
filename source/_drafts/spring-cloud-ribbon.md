---
title: Spring Cloud Ribbon：负载均衡
tags:
  - Spring Cloud
categories: Spring Cloud
---

负载均衡对于一个系统架构来说非常重要，是必须要有的一个基础设施，它能够有效的缓解网络压力和流量扩容。我们知道的负载均衡可以分为两种，一种是硬件的负载均衡，比如 F5 服务器或者 SLB 负载设施；另外一种是软件负载均衡设施，比如我们耳熟能详的 Nginx 反向代理、LVS 负载均衡、HAProxy 负载均衡等等。但是只要是负载均衡服务都是能以类似的架构方式来构建
![负载均衡架构](http://7xq43s.com1.z0.glb.clouddn.com/LoadBalanced.png "负载均衡架构")
<!-- more -->

## Ribbon

Ribbon 是一个客户端负载均衡器，它让你对 HTTP 和 TCP 客户端的行为有更大的控制权。Feign 已经使用了 Ribhon 做负载均衡。当 Ribbon 与 Eureka 一起使用时，`ribbonServerList` 会被 `DiscoveryEnabledNIWSServerList` 重写，扩展成从 Eureka 注册中心中获取服务实例列表。同时它也会用 `NIWSDiscoveryPing` 来取代 `IPing`，它将职责委托给 Eureka 来确定服务端是否已经启动，默认使用轮询访问的规则来达到负载均衡的作用。关于负载均衡可以看看这篇文章 [ Working with load balancers](https://github.com/Netflix/ribbon/wiki/Working-with-load-balancers)。

## RestTemplate

要学习 Ribbon ，我们必须先了解 RestTemplate 的使用，因为 Ribbon 是基于  RestTemplate 模板进行负载调用的。其实跟 HttpClient 差不多，只不过在 RestTemplate 中，基本上都对 GET、PUT、POST、DELETE 进行了重载，提供了更多更简单的调用接口。

### 简单使用

复制 `spring-cloud-01` 的 4 个项目，改为 `spring-cloud-02-ribbon`，再创建一个服务调用方 `request`。
* 注册中心：eureka-a、eureka-b
* 服务提供者：clien-1、client-2
* 服务调用者：request

具体的请到 [GitHub](https://github.com/yangdd1205/spring-cloud-master) 中找以 `spring-cloud-02-ribbon` 开头的项目。 

在 clinet-1 项目中创建
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

我们试试使用 RestTemplate 怎么调用 `sayHello`：
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

首先在 `request` 项目加入 `ribbon` 的依赖。
```XML
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
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



## 示例源码

本篇文章所用示例项目名称均以 `spring-cloud-02-ribbon` 开头

GitHub：https://github.com/yangdd1205/spring-cloud-master

## 参考资料
[Spring Cloud构建微服务架构：服务消费（Ribbon）【Dalston版】](http://blog.didispace.com/spring-cloud-starter-dalston-2-2)
[Spring Cloud Commons: Common Abstractions](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#_spring_cloud_commons_common_abstractions)
[Client Side Load Balancer: Ribbon](http://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#spring-cloud-ribbon)