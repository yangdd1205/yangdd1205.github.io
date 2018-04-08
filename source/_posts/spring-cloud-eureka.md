---
title: Spring Cloud Eureka：服务注册与发现
tags:
  - Spring Cloud
categories: Spring Cloud
date: 2017-12-20 06:44:44
---


Spring Cloud Eureka 是 Spring Cloud Netflix 微服务套件中的一个重要部分。它基于 Netflix Eureka 做了二次封装，主要负责完成微服务架构中的服务治理与服务发现功能。

<!-- more -->

## 环境
* JDK 8
* Maven 3
* IDE：STS or Eclipse 

## 服务注册中心

创建项目名为 `spring-cloud-01-eureka-a`  的 Spring Boot 项目，`pom.xml` 文件如下：
```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.yangdongdong</groupId>
    <artifactId>spring-cloud-01-eureka-a</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>spring-cloud-01-eureka-a</name>
    <description>spring-cloud-01-eureka-a</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.9.RELEASE</version>
        <relativePath /> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Edgware.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>org.yangdongdong.springcloud.EurekaApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

只需要通过注解 `@EnableEurekaServer` 作为 Eureka Server 启动。

```Java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServuerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServuerApplication.class, args);
    }
}
```



在 `application.yml` 修改配置。
```YML
spring:
  application:
    name: eureka-service # 名称 
    
server:
  port: 7001 # 端口   

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false # 是否将自身当作 client 注册，默认值为  true。
    fetch-registry: false # 抓取注册的服务信息，默认值为 true
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

更多的配置选项可以查看 `EurekaInstanceConfigBean` 和 `EurekaClientConfigBean`。

启动项目后，访问：http://localhost:7001 

![Eureka Server](http://p0e1o9bcz.bkt.clouddn.com/eureka/eureka-server.png?	
imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim "注册中心")

可以看到现在还没有服务注册到上面。

![no-instances-available](http://p0e1o9bcz.bkt.clouddn.com/eureka/no-instances-available.png?	
imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim "暂无服务")


## 服务注册

创建一个名为 `spring-cloud-01-client-1` 的 Spring Boot 项目作为 Eureka Client，其 `pom.xml` 如下：
```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.yangdongdong</groupId>
    <artifactId>spring-cloud-01-client-1</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>spring-cloud-01-client</name>
    <description>spring-cloud-01-client-1</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.9.RELEASE</version>
        <relativePath /> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Edgware.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId><!-- 这里有差异 服务端则是  spring-cloud-starter-netflix-eureka-server -->
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>org.yangdongdong.springcloud.ClientApplication</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```
只需要通过注解 `@EnableDiscoveryClient` 表明是一个服务，注册到注册中心上（也可以用注解 `@EnableEurekaClient` 但仅仅只能用于 Eureka）。
```Java
@EnableDiscoveryClient
@SpringBootApplication
public class ClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(ClientApplication.class, args);
    }
}
```

在 `application.yml` 配置 Eureka server 也就是注册中心。
```YML
spring:
  application:
    name: client
server:
  port: 8001
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka/
```

启动项目，再次访问 http://localhost:7001

![instances-available](http://p0e1o9bcz.bkt.clouddn.com/eureka/instances-available.png?	
imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim "Client 已注册")


到这里一个简单版的服务注册和发现就完成了。




## 注册中心以及服务的高可用

在微服务架构环节中，一个非常关键部分是高可用，我们需要充分考虑发生故障的情况。如上面的注册中心是一个单点的服务，如果发生故障，服务则无法注册，也没有办法从注册中心获取服务列表。那么怎么实现注册中心的高可用呢？只需要建立两个 Eureka Server，然后将注册中心自身作为一个服务注册到对方的即可。

由于我们是在单机上做测试，所以需要在 `hosts` 文件中，添加如下两行：
```
127.0.0.1 eureka1
127.0.0.1 eureka2
```

复制项目 `spring-cloud-01-eureka-a` 改为 `spring-cloud-01-eureka-b`，修改 `application.yml`。

eureka-a 的 `application.yml`:
```YML
spring:
  application:
    name: eureka-service # 名称 
    
server:
  port: 7001 # 端口   

eureka:
  instance:
    hostname: eureka1
  client:
    service-url:
      defaultZone: http://eureka2:7002/eureka
```

eureka-b 的 `application.yml`:
```YML
spring:
  application:
    name: eureka-service # 名称 
    
server:
  port: 7002 # 端口   

eureka:
  instance:
    hostname: eureka2
  client:
    service-url:
      defaultZone: http://eureka1:7001/eureka
```

启动 eureka-a 和 eureka-b。

访问 http://eureka1:7001/

![eureka1](http://p0e1o9bcz.bkt.clouddn.com/eureka/eureka1.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim "eureka-a")

访问 http://eureka2:7002/
![eureka2](http://p0e1o9bcz.bkt.clouddn.com/eureka/eureka2.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim "eureka-b")

观察 DS Replicas 和实例 EUREKA-SERVICE 的个数。

 `client-1` 不做任何修改，依然注册到 `eureka-a`，直接启动。访问 http://eureka2:7002/ 会发现 `client-1` 也注册到 `eureka-b` 了。说明互为注册关系的注册中心之间是有同步的。

 不过我们也可以指定 `client-1` 注册到多个注册中心，只需要修改 `application.yml` 中的  `defaultZone`：
 ```YML
 eureka:
  client:
    service-url:
      defaultZone: http://eureka1:7001/eureka/,http://eureka2:7002/eureka/ # 多个注册中心用英文逗号分隔
 ```

而服务的高可用只需要将服务在注册中心注册多次即可。

复制项目 `spring-cloud-01-client-1` 改为 `spring-cloud-01-client-2`，修改端口即可（如果是在不同的机器上部署，可以不用修改端口），启动。

访问 http://eureka1:7001/ 
![client-ha](http://p0e1o9bcz.bkt.clouddn.com/eureka/client-ha.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

可以看到 CLIENT 的个数为 2。

## 服务发现注册机制

![springcloud](http://img.blog.csdn.net/20170727221824930 "图片来自大道化简的博客")


服务注册：服务提供者在启动时会通过发送 Rest 请求的方式将自己注册到 Eureka 服务器上，同时带上自身服务的一些元素。Eureka 收到这个 Rest 请求后，将元数据信息存储在一个双层结构的 Map 中，其中第一层的 key 是服务名，第二层的 key 是具体服务实例名。

服务同步：如架构图所示，这里两个服务提供者分别注册到了两个不同的注册中心上，也就是说他们的信息分别被两个服务注册中心所维护，此时由于服务中心之间因为相互注册服务的关系，当服务提供者发送注册请求到一个服务注册中心的时候，它会将请求转发给集群的其他注册中心，从而实现注册中心之间的服务同步过程。通过服务同步，两个服务提供者的服务信息就可以通过这两台服务注册中心的任意一台获取到。


服务续约：服务注册完成后，服务提供者会维护一个心跳来持续告诉注册中心“我还活在”以防止注册中心“剔除该服务”也就是将服务列表中排除出去。


获取服务：当我们启动消费者的时候，它会发送一个 Rest 请求给注册中心获取上面所有的服务清单，并且为了性能考虑，返回给客户端的清单缓存默认为每 30s 更新一次。


服务调用：消费者获得清单后，通过服务名可以获得具体提供服务的实例名和该实例的元数据信息。因为有这些服务实例的详细信息，所以客户端可以根据自己的需要决定调用哪个服务实例，在 Ribbon 中会默认采用轮训的方式进行调用，从而实现负载均衡机制。

服务下线：在系统运行中必然会出现关闭或重启某个实例的情况，在关闭服务期间，我们自然不希望调用到已经下线的服务实例，所以在客户端程序中，当服务实例正常关闭操作时，它会触发一个服务下线的 Rest 请求给 Eureka，通知其下线，当然注册中心收到该请求后把服务列表状态改为下线，并把该事件传播给集群中其他节点。当出现非正常下线（由于系统内部故障，如内存溢出、服务器宕机等问题）时，注册中心可能并没有收到正确的下线通知请求，而这种情况，Eureka 自己会有一个内部的定时任务，每隔一段时间定时检查超时的清单进行剔除（如果没有续约）。



## 示例源码

本篇文章所用示例项目名称均以 `spring-cloud-01` 开头

GitHub：https://github.com/yangdd1205/spring-cloud-master



## 参考资料

[Eureka Server](https://cloud.spring.io/spring-cloud-static/Edgware.RELEASE/single/spring-cloud.html#spring-cloud-eureka-server)

[Eureka Clients](https://cloud.spring.io/spring-cloud-static/Edgware.RELEASE/single/spring-cloud.html#_service_discovery_eureka_clients)

[Spring Cloud Eureka 详解 - 大道化简](http://blog.csdn.net/sunhuiliang85/article/details/76222517)

