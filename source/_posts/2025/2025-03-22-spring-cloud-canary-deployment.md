---
title: Spring Cloud 全链路灰度发布
date: 2025-03-22 21:42:52
updated: 2025-03-22 21:42:52
tags: [Spring Cloud]
---

在 [Canary Release](https://martinfowler.com/bliki/CanaryRelease.html) 一文中作者说明了灰度发布的概念和流程

{% asset_img canary-release-2.png %}

现在我们来看看在 Spring Cloud 技术体系下该如何实现灰度发布？相关的源码在 [examples/example-cloud](https://github.com/acomma/examples/tree/main/example-cloud)。

<!-- more -->

## 如何标记那些服务是灰度服务？

Eureka 服务注册中心支持元数据

>Additional metadata can be added to the instance registration in the eureka.instance.metadataMap, and this metadata is accessible in the remote clients. In general, additional metadata does not change the behavior of the client, unless the client is made aware of the meaning of the metadata. There are a couple of special cases, described later in this document, where Spring Cloud already assigns meaning to the metadata map.
>
>引用自 [Eureka Metadata for Instances and Clients](https://docs.spring.io/spring-cloud-netflix/reference/spring-cloud-netflix.html#_eureka_metadata_for_instances_and_clients)

因此在发布服务时在 `application.yml` 文件中增加元数据 `eureka.instance.metadata-map.canary`，值为 `true` 时表示该服务是灰度服务

```yaml
eureka:
  instance:
    metadata-map:
      canary: true
```

## 如何标记那些请求是灰度请求？

标记一个请求是灰度请求还是正常请求有很多种方式，比如根据 IP 地址、用户 ID、时区等等来标记。可以在前端发起请求时打标记，也可以在后端入口处打标记。目前我们选择在网关根据登录用户名来标记请求是否为灰度请求，具体实现见 [CustomRequestFilter.java](https://github.com/acomma/examples/blob/main/example-cloud/example-gateway/src/main/java/com/example/gateway/filter/CustomRequestFilter.java)

```java
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // 省略了...

    return reactiveJwtDecoder.decode(accessToken)
            .flatMap(jwt -> {
                ServerHttpRequest newRequest = exchange.getRequest().mutate()
                        // 放入请求头的可以是从 JWT 中获取的用户信息，这里只是简单的把 Subject 信息放进去
                        .header("X-User", jwt.getSubject())
                        .header("X-Canary", isCanary(jwt.getSubject()))
                        .build();
                ServerWebExchange newExchange = exchange.mutate().request(newRequest).build();
                return chain.filter(newExchange);
            })
            .onErrorResume(throwable -> chain.filter(exchange));
}

// 根据用户名判断是否进行灰度
private String isCanary(String subject) {
    return String.valueOf(subject != null && subject.equals("alice"));
}
```

## 灰度请求如何调用灰度服务

Spring Cloud LoadBalancer 允许自定义负载均衡器，参考 [Switching between the load-balancing algorithms](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html#switching-between-the-load-balancing-algorithms) 和 [Passing Your Own Spring Cloud LoadBalancer Configuration](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html#custom-loadbalancer-configuration)。而 Spring Cloud Gateway 默认的负载均衡器是 `RoundRobinLoadBalancer`，它不区分服务是否为灰度服务，为了实现灰度请求调用灰度服务的目的需要参考 `RoundRobinLoadBalancer` 实现 [CanaryRoundRobinLoadBalancer](https://github.com/acomma/examples/blob/main/example-cloud/example-gateway/src/main/java/com/example/gateway/loadbalancer/CanaryRoundRobinLoadBalancer.java)。在这个负载均衡器里需要做两件事情，第一件事是拿到上一步设置在请求头里的灰度标记

```java
public Mono<Response<ServiceInstance>> choose(Request request) {
    List<String> candidates = ((RequestDataContext) request.getContext()).getClientRequest().getHeaders().get("X-Canary");
    String canary = candidates != null && !candidates.isEmpty() ? candidates.getFirst() : "false";

    // 省略了...
}
```

第二件事是把灰度服务和正常服务区分开来，这需要用到服务实例的元数据

```java
private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances, String canary) {
    List<ServiceInstance> normalInstances = new ArrayList<>();
    List<ServiceInstance> canaryInstances = new ArrayList<>();
    for (ServiceInstance instance : instances) {
        if (instance.getMetadata().get("canary") != null && instance.getMetadata().get("canary").equals("true")) {
            canaryInstances.add(instance);
        } else {
            normalInstances.add(instance);
        }
    }
    if (canary.equals("true")) {
        instances = canaryInstances.isEmpty() ? normalInstances : canaryInstances;
    } else {
        instances = normalInstances.isEmpty() ? canaryInstances : normalInstances;
    }

    // 省略了...
}
```

有了负载均衡器后还需要定义它的配置类，注意这个配置类不要添加 `@Configuration` 注解

```java
public class CanaryRoundRobinLoadBalancerClientConfiguration {
    @Bean
    ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(Environment environment, LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new CanaryRoundRobinLoadBalancer(loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class), name);
    }
}
```

接下来通过 `@LoadBalancerClient` 自定义每种服务使用的负载均衡算法

```java
@LoadBalancerClients(
        value = {
                // 使用 Eureka 作为注册中心时 name 必须与 Eureka 中注册的服务名保持一致，Eureka 在注册服务时会把名称转为大写形式，
                // 具体实现为 org.springframework.cloud.netflix.eureka.InstanceInfoFactory 类的 create 方
                @LoadBalancerClient(name = "EXAMPLE-USER", configuration = CanaryRoundRobinLoadBalancerClientConfiguration.class),
                @LoadBalancerClient(name = "EXAMPLE-PRODUCT", configuration = CanaryRoundRobinLoadBalancerClientConfiguration.class),
                @LoadBalancerClient(name = "EXAMPLE-ORDER", configuration = CanaryRoundRobinLoadBalancerClientConfiguration.class),
        }
)
public class ExampleGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExampleGatewayApplication.class, args);
    }
}
```

## 如何实现灰度服务之间的相互调用

首先，定义一个 Feign 拦截器将灰度标记在服务之间进行传递，见 [CustomRequestInterceptor.java](https://github.com/acomma/examples/blob/main/example-cloud/example-order/src/main/java/com/example/order/interceptor/CustomRequestInterceptor.java)

```java
public void apply(RequestTemplate template) {
    ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
    if (requestAttributes != null) {
        HttpServletRequest request = requestAttributes.getRequest();
        // 在实践中这里可以是已授权的用户信息
        String user = request.getHeader("X-User");
        if (user != null) {
            template.header("X-User", user);
        }
        String canary = request.getHeader("X-Canary");
        if (canary != null) {
            template.header("X-Canary", canary);
        }
    }
}
```

其次，定义灰度负载均衡器和它的配置，这部分和网关一样，不再重复。

最后，通过 `@LoadBalancerClient` 自定义每种服务使用的负载均衡算法

```java
@LoadBalancerClients(
        value = {
                // 这里的名字得用小写形式
                @LoadBalancerClient(name = "example-user", configuration = CanaryRoundRobinLoadBalancerClientConfiguration.class),
                @LoadBalancerClient(name = "example-product", configuration = CanaryRoundRobinLoadBalancerClientConfiguration.class),
        }
)
public class ExampleOrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExampleOrderApplication.class, args);
    }
}
```

完~
