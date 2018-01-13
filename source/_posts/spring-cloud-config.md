---
title: Spring Cloud Config：分布式配置中心
tags:
  - Spring Cloud
categories: Spring Cloud
date: 2018-01-13 20:18:56
---


在分布式系统中，配置文件散落在每个项目中，难于集中管理，抑或修改了配置需要重启才能生效。下面我们使用 Spring Cloud Config 来解决这个痛点。

<!-- more -->

## Config Server

我们把 [`config-server`](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-06-config-server) 作为 Config Server，只需要加入依赖：
```XML
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```
在 `application.yml` 中，进行配置：
```YML
spring:
  application:
    name: config-server # 名称 
  cloud:
    config:
      server:
        git:
          uri: file://Users/yangdd/Documents/code/GitHub/config-repo # 使用本地仓库（用于测试），格式为：file://${user.home}/config-repo 如果是 Windows 则是 file:///${user.home}/config-repo
          #uri: https://github.com/yangdd1205/spring-cloud-master/
          #username: 用户名
          #password: 密码
          #default-label: config # 可以是 commit id，branch name, tag name，默认值为 master
          #search-paths: # 表示除了在根目录搜索配置文件，还可以在以下目录搜索
          #- user          
server:
  port: 4001 # 端口   
```

Spring 的项目，一般有个“约定大于配置”的原则。所以我们 Git 中的配置文件，一般以 `{application}-{profile}.yml` 或者 `{application}-{profile}.properties` 命名。
* `{appliction}` 映射到客户端 `spring.application.name` 属性值
* `{profile}` 映射到客户端 `spring.profiles.active` 属性值

我们在 `config-repo` 的 `master` 分支中，有两个文件
`client-dev.yml`：
```YML
info: master-dev
```
`client-prod.yml`：
```YML
info: master-prod
```
在 `config` 分支中，有两个文文件
`client-dev.yml`：
```YML
info: config-dev
```
`client-prod.yml`：
```YML
info: config-prod
```

启动 `config-server` 我们可以通过下面的映射关系来访问 Git 中的配置文件：
```
/{application}/{profile}[/{label}]
/{application}-{profile}.yml
/{label}/{application}-{profile}.yml
/{application}-{profile}.properties
/{label}/{application}-{profile}.properties
```
http://localhost:4001/client-dev.yml

## Config Client

我们把 [`config-client`](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-06-config-client) 作为 Config Client，加入依赖：
```XML
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
  </dependency>
```
在 `bootstrap.yml` 中配置（`bootstrap.yml` 会先于 `application.yml` 加载）：
```YML
spring:
  application: 
    name: client
  cloud:
    config:
      uri: http://localhost:4001/ ##表示配置中心的地址
      profile: dev # 等同于 spring.profiles.active 属性，但是优先级高于 spring.profiles.active
      #label: config # 如果不指定使用 server 中的 label，如果指定了则以 client 的为准
    
server:
  port: 4002 # 端口   
```

在 `config-client` 创建 `TestController`：
```Java
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

访问 http://localhost:4002/getInfo 

修改 `config-client` 配置文件的中 `profile` 为 `prod` 再访问，查看结果。

现在我们已经实现了，配置文件统一管理，但是修改配置文件后，如何不重启生效呢？

我们在 `config-client` 中引入依赖：
```XML
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
在需要动态刷新的类上加上注解 `@RefreshScope`，这里我们在 `TestController` 加上。并在配置文件中把 actuator 的权限认证禁用：
```YML
management:
  security:
    enabled: false
```

重启项目，http://localhost:4002/getInfo 结果为：`config-dev`，现在修改 `config-dev.yml` 文件：
```YML
info: config-dev-1.0
```

调用 actuator 的方法，手动刷新，http://localhost:4002/refresh **注意：必须是 POST 方式。 ** 可以看到发生改变了的值有哪些：
```JSON
[
    "info"
]
```

再访问 http://localhost:4002/getInfo 结果为：`config-dev-1.0`。

我们发现现在虽然可以动态刷新配置了，但是每次都要手动调用刷新方法，假如我有 1000 个服务，那就要调用 1000 次，有没有更好的办法呢？那就是结合 Spring Cloud Bus 使用。

## 高可用

现在我们的 Config Server 是一个单点的服务，一旦挂掉后面的 Config Client 将无法使用。所以在生成中，Config Server 必须是高可用的，那怎么做呢？很简单，跟我们前面讲的 Eureka 的高可用一样，只需要将同一个 Config Server 多部署几个即可。

我们将项目 [`eureka`](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-06-config-eureka) 作为注册中心，端口为：7001。

复制 `config-server` 改为 [`config-server-1`](https://github.com/yangdd1205/spring-cloud-master/tree/master/spring-cloud-06-config-server-1)，端口为：4003，将两个 Config Server 都注册到注册中心。

将 `config-client` 也注册到注册中心，并指定从 Eureka 中获取 Config Server：
```YML
spring:
  cloud:
    config:
      #uri: http://localhost:4001/ ##表示配置中心的地址，直接指定 Config Server 地址 
      profile: dev
      #label: config
      discovery:
        enabled: true # 启用从服务中获取 Config Server
        service-id: config-server # Config Server 的服务名
```


## 示例源码

本篇文章所用示例项目名称均以 `spring-cloud-06-config` 开头

GitHub：https://github.com/yangdd1205/spring-cloud-master

## 参考资料

[Spring Cloud Config](http://cloud.spring.io/spring-cloud-static/spring-cloud-config/1.4.0.RELEASE/single/spring-cloud-config.html)

