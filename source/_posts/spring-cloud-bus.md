---
title: Spring Cloud Bus：消息总线
tags:
  - Spring Cloud
categories: Spring Cloud
date: 2018-01-21 13:49:37
---

在上一篇中，我们使用 Spring Cloud Config 来实现了，分布式配置中心以及动态更新，但是仍然有一个问题，每次更新都需要每个 Config Client 手动调用 `/refresh` 刷新，这太麻烦了，下面我们就用 Spring Cloud Bus 来解决这个问题。
<!-- more-->

## Spring Cloud Bus

Spring Cloud Bus 将分布式系统的结点与轻量级消息链接起来，可以用于广播状态改变（例如配置改变）或者其他管理指令。

也就是说，配置仓库发生改变了，我们可以通过 Config Server 发起广播通知到各个 Config Client 重新获取配置信息。


这里我们使用 [Kafka](http://kafka.apache.org/) 来作为消息代理。具体的安装可以查看[官网文档](http://kafka.apache.org/quickstart)

{% note info %} 
在启动 Kafka 时，如果出现 `java.net.UnknownHostException` 异常，有两种办法：
1. 在 `{kafkapath}/config/server.properties` 中添加 `host.name=192.168.222.130`
2. 在 `/etc/hosts` 文件中添加 `127.0.0.1 主机名`
 {% endnote %}

基于上一篇文章的例子，我们创建 [`bus-server`](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-07-bus-server) 和 [`bus-client`](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-07-bus-client)

在 `bus-server` 中添加依赖：
```XML
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
</dependencies>
```
其 `application.yml` 配置如下：
```YML
spring:
  application:
    name: config-server # 名称 
  cloud:
    config:
      server:
        git:
          uri: file://Users/yangdd/Documents/code/GitHub/config-repo 
          default-label: config 
    stream:
      kafka: 
        binder:
          zk-nodes: 192.168.222.130:2181
          brokers: 192.168.222.130:9092        
management:
  security:
    enabled: false   
server:
  port: 4001 # 端口   
```

在 `bus-client` 中引入依赖：
```XML
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-config</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>
</dependencies>
```
其 `bootstrap.yml` 如下：
```YML
spring:
  application: 
    name: client
  cloud:
    config:
      uri: http://localhost:4001/
      profile: dev
    stream:
      kafka:
        binder:
          zk-nodes: 192.168.222.130:2181
          brokers: 192.168.222.130:9092
        
management:
  security:
    enabled: false
server:
  port: 4002 # 端口   
```

写个测试方法：
```Java
@RefreshScope 
@RestController
public class TestController {

    @Value("${info}")
    private String info;

    @RequestMapping(value = "/getInfo")
    public String getInfo() {
        return this.info;
    }

}
```

依次启动 `bus-server` 和 `bus-client`。

此时，我们可以使用 `kafka-topics --list --zookeeper localhost:2181` 命令来查看当前 Kafka 中的 Topic，如果项目成功启动则会发现一个名为 `springCloudBus` 的 Topic,这个就是 Spring Cloud Bus 的。


现在，访问 http://localhost:4002/getInfo 现在是 `config-dev-1.0`。

修改配置仓库中 `client-dev.yml` 文件 `info: config-dev-2.0`。

访问 http://localhost:4001/client/dev 发现配置文件已修改，现在通过 Config Server 发送广播，其实就是通过 `POST` 方式请求 http://localhost:4001/bus/refresh，再次访问 http://localhost:4002/getInfo 现在是 `config-dev-2.0`。

Spring Cloud Bus 还支持指定刷新范围，在某些特殊场景（比如：灰度发布），我们希望可以刷新微服务中某个具体实例的配置，则可以通过 `/bus/refresh` 接口的 `destination` 来指定刷新的应用，比如：`/bus/refresh?destination=customers:9000` 则表示实例名 `spring.application.name` 为 `customers`，端口 `server.port` 为 `9000` 的应用程序会刷新配置。当然还可以通过 `/bus/refresh?destination=customers:**` 指定实例名为 `customers` 的所有应用程序。

## 示例源码

本篇文章所用示例项目名称均以 `spring-cloud-07-bus` 开头

GitHub：https://github.com/yangdd1205/spring-cloud-master

## 参考资料
[Spring Cloud Bus](http://cloud.spring.io/spring-cloud-static/spring-cloud-bus/1.3.2.RELEASE/single/spring-cloud-bus.html)

[Spring Cloud 构建微服务架构（七）消息总线（Rabbit）](http://blog.didispace.com/springcloud7/)

[Spring Cloud 构建微服务架构（七）消息总线（Kafka）](http://blog.didispace.com/springcloud7-2/)
