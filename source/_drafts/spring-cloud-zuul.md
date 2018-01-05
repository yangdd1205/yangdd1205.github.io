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
    * `static`：特殊的 Filter 具体的可以看 StaticResponseFilter，它允许从 Zuul 本身生成响应，而不是将请求转发到源。
* `filterOrder`：处理的优先级，值越小表示优先级越高
* `shouldFilter`：该 filter 是否执行
* `run`：filter 具体的业务逻辑

我们现在写一个校验 TOKEN 的过滤器：
```Java
@Component // 交由 Spring 管理
public class CustomAuthFilter extends ZuulFilter {

    private static final String TOKEN_AUTH = "123456";
    
    /**
     * 指定服务名字
     */
    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        String uri = request.getRequestURI();
//        if (uri.equals("/hi-service/upload") || uri.equals("/zuul/hi-service/upload")) {
//            return ctx;
//        }
        String token = request.getHeader("x-auth-token");

        if (StringUtils.isEmpty(token)) {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            ctx.setResponseBody("no token");
            return null;
        }
        if(TOKEN_AUTH.equals(token)) {
            ctx.addZuulRequestHeader("userInfo", "{\"name\":\"Tom\",\"age\":18}");
        }else {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
            ctx.setResponseBody("token auth fail");
            return null; 
        }
        return ctx;
    }

    /**
     * filter 是否执行
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    /**
     * filter 的优先级，值越小优先级越高
     */
    @Override
    public int filterOrder() {
        return 0;
    }

    /**
     * filter 执行的时机
     *
     * pre 表示在请求被路由之前执行
     * 
     * route 表示在请求被路由中执行
     * 
     * post 表示在请求被路由之后执行
     * 
     * error 表示在请求发生错误时执行
     * 
     * static 特殊的 Filter 具体的可以看 StaticResponseFilter，它允许从 Zuul 本身生成响应，而不是将请求转发到源。
     */
    @Override
    public String filterType() {
        return "pre";
    }

}
```

现在再通过网关请求服务就需要在 header 里面提供 x-auth-token 了。可以使用 Postman 来测试。

我们在 `HelloController` 里面再写一个方法试试能否获取到 Zuul 添加在 header 里面的 userInfo：
```Java
@RequestMapping(value = "/userInfo", method = RequestMethod.GET)
public String auth(@RequestHeader("userInfo") String userInfo, String id) {
    System.out.println("id:" + id);
    return userInfo;
}
```

Zuul 还支持在某个服务发生 fallback 时，自定义返回值，只需要实现 `FallbackProvider` 接口即可。
```Java
package org.yangdongdong.springcloud;

@Component // 交由 Spring 管理
public class HiServiceZuulFallBackProvider implements FallbackProvider {

    /**
     * 指定服务名字
     */
    @Override
    public String getRoute() {
        return "hi";
    }

    @Override
    public ClientHttpResponse fallbackResponse() {
        return new ClientHttpResponse() {

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON_UTF8);
                return null;
            }

            @Override
            public InputStream getBody() throws IOException {
                String result = "{\"result:\":\"bad request\"}";
                return new ByteArrayInputStream(result.getBytes("UTF-8"));
            }

            /**
             * 状态文本信息
             */
            @Override
            public String getStatusText() throws IOException {
                return getStatusCode().getReasonPhrase();
            }

            /**
             * 自定义响应的状态
             */
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.BAD_REQUEST;
            }

            /**
             * 自定义响应的状态码
             */
            @Override
            public int getRawStatusCode() throws IOException {
                return getStatusCode().value();
            }

            @Override
            public void close() {

            }
        };
    }

    @Override
    public ClientHttpResponse fallbackResponse(Throwable cause) {
        return new ClientHttpResponse() {

            @Override
            public HttpHeaders getHeaders() {
                HttpHeaders headers = new HttpHeaders();
                headers.setContentType(MediaType.APPLICATION_JSON_UTF8);
                return headers;
            }

            @Override
            public InputStream getBody() throws IOException {
                String result = "{\"result:\":\"error\"}";
                return new ByteArrayInputStream(result.getBytes("UTF-8"));
            }

            /**
             * 状态文本信息
             */
            @Override
            public String getStatusText() throws IOException {
                return getStatusCode().getReasonPhrase();
            }

            /**
             * 自定义响应的状态
             */
            @Override
            public HttpStatus getStatusCode() throws IOException {
                return HttpStatus.INTERNAL_SERVER_ERROR;
            }

            /**
             * 自定义响应的状态码
             */
            @Override
            public int getRawStatusCode() throws IOException {
                return getStatusCode().value();
            }

            @Override
            public void close() {

            }
        };
    }

}
```

我们把 `hi-service` 停掉，访问 http://localhost:6001/hi-service/hi 


## 文件上传

我们现在写一个文件上传的功能，在 `HiController` 中加入文件上传的方法：
```Java
@RequestMapping(value="/upload",method=RequestMethod.POST)
public String upload(@RequestParam("file")MultipartFile file) {
    System.out.println("名称："+file.getOriginalFilename());
    try {
        FileCopyUtils.copy(file.getBytes(),new File(file.getOriginalFilename()));
        return "success";
    } catch (IOException e) {
        e.printStackTrace();
        return "error";
    }
    
}
```

然后再配置文件中添加配置：
```YML
spring:
  application:
    name: hi
  http:
    encoding:
      charset: UTF-8
    multipart:
      enabled: true
      file-size-threshold: 10
      max-file-size: 50MB
      max-request-size: 80MB
```

在 `resources` 文件夹下，创建 `META-INF/resources/index.html`：
```HTML
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Insert title here</title>
</head>
<body>
	<div>
		<form action="http://localhost:8002/upload"
			enctype="multipart/form-data" method="post">
			<input type="file" name="file"> <br /> <input type="submit"
				value="直接通过 hi 服务上传">
		</form>
	</div>
	<div>
		<form action="http://localhost:6001/hi-service/upload"
			enctype="multipart/form-data" method="post">
			<input type="file" name="file"> <br /> <input type="submit"
				value="通过 gateway 上传">
		</form>
	</div>
	<div>
		<form action="http://localhost:6001/zuul/hi-service/upload"
			enctype="multipart/form-data" method="post">
			<input type="file" name="file"> <br /> <input type="submit"
				value="绕过 zuul 的 dispatchServlet 进行上传">
		</form>
	</div>
</body>
</html>
```

注意，记得在 `CustomAuthFilter` 中，加上 文件上传的 URI 不校验 TOKEN。

第一个 `form` 是通过 `hi` 服务进行上传，当文件小于 50M 的时候，没有问题。

第二个 `form` 是通过 `gateway` 服务进行上传，但是当文件大约 10M 的时候就会报错：
```Java
org.apache.tomcat.util.http.fileupload.FileUploadBase$SizeLimitExceededException: the request was rejected because its size (26684755) exceeds the configured maximum (10485760)
```

原来 Zuul 有自己的 `DispatchServlet`，我们可以使用 `/zuul` 前缀来表示绕过 Zuul 内部的 `DispatchServlet` 直接路由到具体的服务，这样可以避免 2 次配置，并且非常的方便，只要符合 `/zuul/*` 的配置即可。第三个 `form` 既是如此。


## 示例源码

本篇文章所用示例项目名称均以 `spring-cloud-05-zuul` 开头

GitHub：https://github.com/yangdd1205/spring-cloud-master
