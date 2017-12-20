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

{% blockquote Spring.io http://cloud.spring.io/spring-cloud-static/Dalston.SR4/multi/multi_spring-cloud-ribbon.html %}

Ribbon is a client side load balancer which gives you a lot of control over the behaviour of HTTP and TCP clients. Feign already uses Ribbon, so if you are using @FeignClient then this section also applies.

{% endblockquote %}

Ribbon 是一个客户端负载均衡器，它让你对 HTTP 和 TCP 客户端的行为有更大的控制权。Feign 已经使用了 Ribhon 做负载均衡。当 Ribbon 与 Eureka 一起使用时，`ribbonServerList` 会被 `DiscoveryEnabledNIWSServerList` 重写，扩展成从 Eureka 注册中心中获取服务实例列表。同时它也会用 `NIWSDiscoveryPing` 来取代 `IPing`，它将职责委托给 Eureka 来确定服务端是否已经启动。

### RestTemplate
{% blockquote %}
RestTemplate can be automatically configured to use ribbon. To create a load balanced RestTemplate create a RestTemplate @Bean and use the @LoadBalanced qualifier.
{% endblockquote%}

要学习 Ribbon ，我们必须先了解 RestTemplate 的使用，因为 Ribbon 是基于  RestTemplate 模板进行负载调用的。

### 参考资料
[Working with load balancers](https://github.com/Netflix/ribbon/wiki/Working-with-load-balancers)
[Spring Cloud构建微服务架构：服务消费（Ribbon）【Dalston版】](http://blog.didispace.com/spring-cloud-starter-dalston-2-2)