---
title: Spring Cloud Eureka：服务注册与发现
tags:
  - Spring Cloud
categories: Spring Cloud
---

Spring Cloud Eureka 是 Spring Cloud Netflix 微服务套件中的一个重要部分。它基于 Netflix Eureka 做了二次封装，主要负责完成微服务架构中的服务治理与服务发现功能。

<!-- more -->

## 环境
* JDK 8
* Maven 3.5.0
* IDE：IDEA or STS

## 服务注册中心

创建项目名为 `spring-cloud-01-eureka-a`  的 Spring Boot 项目，在 pom.xml 文件如下：
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
			<artifactId>spring-cloud-starter-eureka-server</artifactId>
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

只需要通过注解 `@EnableEurekaServer` 启用 Eureka 注册中心服务

```Java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServuerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServuerApplication.class, args);
	}
}
```


在 `application.yml` 修改配置
```YML

```

启动项目后，访问：http://localhost:7001 

![Eureka Server]()


## 服务注册


## 更多

### 注册中心的高可用


### 启用用户名和密码





## 参考资料

Eureka Server：https://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#spring-cloud-eureka-server

Eureka Clients：https://cloud.spring.io/spring-cloud-static/Dalston.SR4/single/spring-cloud.html#_service_discovery_eureka_clients

