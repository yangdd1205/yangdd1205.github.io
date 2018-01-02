---
title: Spring Cloud Zuul：网关
tags:
  - Spring Cloud
categories: Spring Cloud
---
网关顾名思义就是网络请求的入口，外部服务通过网关进行访问我们的内部服务，网关可以做很多的事，进行限流、安全、权限认证、黑白名单过滤、请求拦截、数据校验、API 路由转发等等功能。

<!-- more -->

## Zuul 

服务网关是微服务架构中一个不可或缺的部分，通过服务网关统一向外系统提供 REST API 的过程中，除了具备服务路由、负载均衡功能之外，它还具备了权限控制的功能。Spring Cloud Netflix 中的 Zuul 就担任了这样一个角色，为微服务架构提供了前门保护的作用，同时将权限控制这些比较重的非业务逻辑内容迁移到服务路由层面，使得服务集群主体能够具备更高的可复用性和可测试性。


## 使用

现有四个项目如下：
* [server](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-05-zuul-server)：注册中心，服务名：eureka-server，端口：7001
* [hello-service](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-05-zuul-hello-service)：服务提供者，服务名：helllo，端口：8001
* [hi-service](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-05-zuul-hi-service)：服务提供者，服务名：hi，端口：8002
* [gateway](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-05-zuul-gateway)：服务网关，服务名：gateway，端口：6001

`hello-service` 提供如下服务：
```Java
@RestController
public class HelloController {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String sayHello() {
        System.out.println("hello service");
        return "hello!";
    }
    
}
```
`hi-service` 提供如下服务：
```Java
@RestController
public class HiController {

    @RequestMapping(value = "/hi", method = RequestMethod.GET)
    public String sayHello() {
        System.out.println("hi service");
        return "hi!";
    }

}
```
在 `gateway` 引入 Zuul 依赖：
```XML
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

依次启动项目，现在我们通过网关来访问服务：http://localhost:6001/hello/hello http://localhost:6001/hi/hi

如果我们什么都不配置的话，Zuul 会使用默认路由规则，即端口后为服务名，如：http://localhost:6001/hello/hello 表示调用 `hello` 服务的 `hello` 方法。   

当然我们也可以自定义 Zuul 的路由规则，修改 `gateway` 的配置文件：
```YML
zuul:
  routes:
    api-hello:
      path: /hello-service/**  
      service-id: hello
    api-hi:
      path: /hi-service/**
      service-id: hi  
```

现在的访问路径就变成了，http://localhost:6001/hello-service/hello http://localhost:6001/hi-service/hi

通过 Zuul 组件的 jar 可以发现，它所依赖的组件有 Ribbon 和 Hystrix，这说明 Zuul 同样集成了这两个组件，同样能够进行负载均衡和断路器功能的实现。

我们在网关上设置全局的 Ribbon 和 Hystrix 超时时间都要尽可能的大，保证包含内部的超时、重试、熔断、降级的时间。当然 Zuul 内部本身也有熔断功能。

## 过滤器

使用 Zuul 过滤器，我们通过 Zuul 的过滤器可以对请求的一些信息进行安全认证、权限认证、身份识别等功能。只需要自定义 `Filter` 后继承 `ZuulFilter` 类，重写下面方法即可：
* `filterType`：filter 执行的时机
    * `pre`：在请求被路由之前执行
    * `route`：在请求被路由时执行
    * `post`：在请求被路由之后执行
    * `error`：在请求发生错误时执行
    * `static`：暂不知道
* `filterOrder`：处理的优先级，值越小表示优先级越高
* `shouldFilter`：该 filter 是否执行
* `run`：filter 具体的业务逻辑


