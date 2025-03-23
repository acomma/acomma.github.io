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

## 答疑时间

### 为什么在网关自定义负载均衡时名称需要大写？

#### 服务提供方注册大写形式的应用名称

>The default application name (that is, the service ID), virtual host, and non-secure port (taken from the `Environment`) are `${spring.application.name}`, `${spring.application.name}` and `${server.port}`, respectively.
>
>Having `spring-cloud-starter-netflix-eureka-client` on the classpath makes the app into both a Eureka “instance” (that is, it registers itself) and a “client” (it can query the registry to locate other services). The instance behaviour is driven by `eureka.instance.*` configuration keys, but the defaults are fine if you ensure that your application has a value for `spring.application.name` (this is the default for the Eureka service ID or VIP).
>
>引用自 [Registering with Eureka](https://docs.spring.io/spring-cloud-netflix/reference/spring-cloud-netflix.html#_registering_with_eureka)

当使用 Eureka 作为服务注册中心时，Eureka 默认将 `spring.application.name` 的值作为应用名称或服务 ID，比如 `example-user`。我们配置名称都是小写形式，但是在网关使用 `@LoadBalancerClient` 注解自定义服务负载均衡时要求 `name` 属性的值必须使用大写形式，比如 `EXAMPLE-USER`，这是为什么呢？

Eureka 客户端在启动时通过 `EurekaClientAutoConfiguration` 向容器中注入了 `ApplicationInfoManager`，这个 Bean 在实例化时调用 `InstanceInfoFactory` 类的 `create` 方法把配置信息转换成 `InstanceInfo` 对象

```java
public InstanceInfo create(EurekaInstanceConfig config) {
    // 省略了...

    InstanceInfo.Builder builder = InstanceInfo.Builder.newBuilder();

    // 省略了...

    builder.setNamespace(namespace)
        .setAppName(config.getAppname())
        .setInstanceId(config.getInstanceId())

    // 省略了...
}
```

注意到第 9 行的 `setAppName` 方法，就是在这里将应用名称强制转换为了大写形式

```java
public Builder setAppName(String appName) {
    this.result.appName = (String)this.intern.apply(appName.toUpperCase(Locale.ROOT));
    return this;
}
```

最终 `com.netflix.discovery.DiscoveryClient` 的 `register` 方法使用前面创建的 `InstanceInfo` 对象向服务注册中心注册服务。这部分是服务提供方的服务注册逻辑，下面来看看服务消费方的服务发现逻辑。

#### 服务消费方构建大写形式的路由

在配置 `spring.cloud.gateway.discovery.locator.enabled` 的值为 `true` 时 `GatewayDiscoveryClientAutoConfiguration` 向容器中注入 `DiscoveryClientRouteDefinitionLocator`，这个 Bean 在实例化时从服务注册中心获取 `ServiceInstance`

```java
public DiscoveryClientRouteDefinitionLocator(ReactiveDiscoveryClient discoveryClient,
        DiscoveryLocatorProperties properties) {
    this(discoveryClient.getClass().getSimpleName(), properties);
    serviceInstances = discoveryClient.getServices()
        .flatMap(service -> discoveryClient.getInstances(service).collectList());
}
```

在使用 Eureka 作为服务注册中心时 `ServiceInstance` 的实现类是 `EurekaServiceInstance`，这个类有一个成员变量 `InstanceInfo`，这个变量的内容和服务提供方注册的内容一致。

网关收到请求后 `RoutePredicateHandlerMapping` 的 `getHandlerInternal` 方法调用 `lookupRoute` 方法获取路由 `Route`，经过一系列代理后调用 `RouteDefinitionRouteLocator` 的 `getRoutes` 方法获取路由。`RouteDefinitionRouteLocator#getRoutes` 方法里调用 `DiscoveryClientRouteDefinitionLocator#getRouteDefinitions` 方法把 `ServiceInstance` 转换为 `RouteDefinition` 

```java
protected RouteDefinition buildRouteDefinition(Expression urlExpr, ServiceInstance serviceInstance) {
    String serviceId = serviceInstance.getServiceId();
    RouteDefinition routeDefinition = new RouteDefinition();
    routeDefinition.setId(this.routeIdPrefix + serviceId);
    String uri = urlExpr.getValue(this.evalCtxt, serviceInstance, String.class);
    routeDefinition.setUri(URI.create(uri));
    // add instance metadata
    routeDefinition.setMetadata(new LinkedHashMap<>(serviceInstance.getMetadata()));
    return routeDefinition;
}
```

方法参数中 `urlExpr` 变量的格式为 `'lb://'+serviceId`，第 2 行 `serviceInstance.getServiceId()` 获取的是 `InstanceInfo#appName` 变量的值，这个值为大写形式的应用名称，比如 `EXAMPLE-USER`，因此第 5 行得到的 `uri` 为 `lb://EXAMPLE-USER`，这个值会设置到 `Route#uri` 变量中，即使用大写形式的应用名称作为路由 URI 的 `scheme` 部分。路由构建好后在 `RoutePredicateHandlerMapping#getHandlerInternal` 中将路由设置到 `ServerWebExchange` 的属性中 `exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);`。

#### 路由转换为请求地址

在 `RouteToRequestUrlFilter` 中将路由 URI 和实际请求的 URI 进行合并

```java
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    Route route = exchange.getAttribute(GATEWAY_ROUTE_ATTR);
    
    // 省略了...

    URI uri = exchange.getRequest().getURI();
    boolean encoded = containsEncodedParts(uri);
    URI routeUri = route.getUri();

    //  省略了...

    URI mergedUrl = UriComponentsBuilder.fromUri(uri)
        // .uri(routeUri)
        .scheme(routeUri.getScheme())
        .host(routeUri.getHost())
        .port(routeUri.getPort())
        .build(encoded)
        .toUri();
    exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, mergedUrl);
    return chain.filter(exchange);
}
```

合并的逻辑是用路由 URI 的 `scheme`、`host`、`port` 替换实际请求的 URI 的对应部分，比如请求的 URI 为 `http://localhost:8080/user/1`，路由的 URI 为 `lb://EXAMPLE-USER`，合并后的结果为 `lb://EXAMPLE-USER/user/1`。最后把合并后的 URI 设置到 `ServerWebExchange` 的 `GATEWAY_REQUEST_URL_ATTR` 属性中。

#### 创建特定于每个应用名称的 Bean 容器

在 `ReactiveLoadBalancerClientFilter` 中触发为每个应用名称创建独立的 Bean 容器

```java
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
    // 省略了...
    // preserve the original url
    addOriginalRequestUrl(exchange, url);
    // 省略了...
    URI requestUri = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
    String serviceId = requestUri.getHost();
    Set<LoadBalancerLifecycle> supportedLifecycleProcessors = LoadBalancerLifecycleValidator
        .getSupportedLifecycleProcessors(clientFactory.getInstances(serviceId, LoadBalancerLifecycle.class),
                RequestDataContext.class, ResponseData.class, ServiceInstance.class);
    DefaultRequest<RequestDataContext> lbRequest = new DefaultRequest<>(new RequestDataContext(
            new RequestData(exchange.getRequest(), exchange.getAttributes()), getHint(serviceId)));
    return choose(lbRequest, serviceId, supportedLifecycleProcessors).doOnNext(response -> {
    // 省略了...
}
```

第 7 行从 `ServerWebExchange` 中取出 `GATEWAY_REQUEST_URL_ATTR` 属性的值，即 `lb://EXAMPLE-USER/user/1`。第 8 行从 URI 变量中去除 HOST，即 `EXAMPLE-USER`，作为服务的 ID（应用名称）。第 10 行的 `clientFactory.getInstances` 在第一次调用时将触发创建特定于每个应用名称的 Bean 容器的过程

```java
// LoadBalancerClientFactory.java 的父类方法
public GenericApplicationContext createContext(String name) {
    GenericApplicationContext context = buildContext(name);
    // there's an AOT initializer for this context
    if (applicationContextInitializers.get(name) != null) {
        applicationContextInitializers.get(name).initialize(context);
        context.refresh();
        return context;
    }
    registerBeans(name, context);
    context.refresh();
    return context;
}
```

我们将重点看看第 10 行的 `registerBeans` 方法

```java
public void registerBeans(String name, GenericApplicationContext context) {
    Assert.isInstanceOf(AnnotationConfigRegistry.class, context);
    AnnotationConfigRegistry registry = (AnnotationConfigRegistry) context;
    if (this.configurations.containsKey(name)) {
        for (Class<?> configuration : this.configurations.get(name).getConfiguration()) {
            registry.register(configuration);
        }
    }
    for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
        if (entry.getKey().startsWith("default.")) {
            for (Class<?> configuration : entry.getValue().getConfiguration()) {
                registry.register(configuration);
            }
        }
    }
    registry.register(PropertyPlaceholderAutoConfiguration.class, this.defaultConfigType);
}
```

`this.configurations` 包含所有的 `LoadBalancerClientSpecification` 类型的 Bean，通过 `@LoadBalancerClient` 注解定义的也包含在内。第 4~8 行会注入我们定义的配置类 `CanaryRoundRobinLoadBalancerClientConfiguration`，从而覆盖第 16 行注入的 `this.defaultConfigType`，即 `LoadBalancerClientConfiguration` 配置类。这就是在网关自定义负载均衡时名称需要大写的原因。

### 为什么在 Feign 客户端自定义负载均衡时名称需要小写？

#### 构建 Feign 客户端调用地址

在项目启动时 `FeignClientsRegistrar` 扫描 `@FeignClient` 注解标注的类注册 Feign 客户端

```java
public void registerFeignClients(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    LinkedHashSet<BeanDefinition> candidateComponents = new LinkedHashSet<>();
    Map<String, Object> attrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName());
    final Class<?>[] clients = attrs == null ? null : (Class<?>[]) attrs.get("clients");
    if (clients == null || clients.length == 0) {
        ClassPathScanningCandidateComponentProvider scanner = getScanner();
        scanner.setResourceLoader(this.resourceLoader);
        scanner.addIncludeFilter(new AnnotationTypeFilter(FeignClient.class));
        Set<String> basePackages = getBasePackages(metadata);
        for (String basePackage : basePackages) {
            candidateComponents.addAll(scanner.findCandidateComponents(basePackage));
        }
    }
    else {
        for (Class<?> clazz : clients) {
            candidateComponents.add(new AnnotatedGenericBeanDefinition(clazz));
        }
    }

    for (BeanDefinition candidateComponent : candidateComponents) {
        if (candidateComponent instanceof AnnotatedBeanDefinition beanDefinition) {
            // verify annotated class is an interface
            AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
            Assert.isTrue(annotationMetadata.isInterface(), "@FeignClient can only be specified on an interface");

            Map<String, Object> attributes = annotationMetadata
                .getAnnotationAttributes(FeignClient.class.getCanonicalName());

            String name = getClientName(attributes);
            String className = annotationMetadata.getClassName();
            registerClientConfiguration(registry, name, className, attributes.get("configuration"));

            registerFeignClient(registry, annotationMetadata, attributes);
        }
    }
}
```

在当前使用 Feign 的方式下，即 `@FeignClient(name = "example-user")`，只需要注意第 33 行的 `registerFeignClient` 方法调用，在未配置 `spring.cloud.openfeign.lazy-attributes-resolution` 属性或者其值为 `false` 时将调用 `eagerlyRegisterFeignClientBeanDefinition` 方法注册 `FeignClientFactoryBean`

```java
private void eagerlyRegisterFeignClientBeanDefinition(String className, Map<String, Object> attributes,
        BeanDefinitionRegistry registry) {
    validate(attributes);
    BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(FeignClientFactoryBean.class);
    // 省略了...
    String name = getName(attributes);
    definition.addPropertyValue("name", name);
    String contextId = getContextId(null, attributes);
    definition.addPropertyValue("contextId", contextId);
    // 省略了...
}
```

这里需要注意第 5~9 行的 `name` 和 `contextId` 的值，在当前使用 Feign 的方式下它们的值都是 `example-user`。`FeignClientFactoryBean` 的 `getTarget` 方法会构建调用的 `url`

```java
<T> T getTarget() {
    FeignClientFactory feignClientFactory = beanFactory != null ? beanFactory.getBean(FeignClientFactory.class)
            : applicationContext.getBean(FeignClientFactory.class);
    Feign.Builder builder = feign(feignClientFactory);
    if (!StringUtils.hasText(url) && !isUrlAvailableInConfig(contextId)) {
        // 省略了...
        if (!name.startsWith("http://") && !name.startsWith("https://")) {
            url = "http://" + name;
        }
        else {
            url = name;
        }
        url += cleanPath();
        return (T) loadBalance(builder, feignClientFactory, new HardCodedTarget<>(type, name, url));
    }
    // 省略了...
}
```

这个方法构建的 `url` 为 `http://example-user`，域名是在 `@FeignClient` 注解定义的名称。

#### 创建特定于每个 Feign 客户端的 Bean 容器

通过 Feign 客户端调用远程服务会进入 `FeignBlockingLoadBalancerClient` 的 `execute` 方法

```java
public Response execute(Request request, Request.Options options) throws IOException {
    final URI originalUri = URI.create(request.url());
    String serviceId = originalUri.getHost();
    Assert.state(serviceId != null, "Request URI does not contain a valid hostname: " + originalUri);
    String hint = getHint(serviceId);
    DefaultRequest<RequestDataContext> lbRequest = new DefaultRequest<>(
            new RequestDataContext(buildRequestData(request), hint));
    Set<LoadBalancerLifecycle> supportedLifecycleProcessors = LoadBalancerLifecycleValidator
        .getSupportedLifecycleProcessors(
                loadBalancerClientFactory.getInstances(serviceId, LoadBalancerLifecycle.class),
                RequestDataContext.class, ResponseData.class, ServiceInstance.class);
    supportedLifecycleProcessors.forEach(lifecycle -> lifecycle.onStart(lbRequest));
    ServiceInstance instance = loadBalancerClient.choose(serviceId, lbRequest);
    org.springframework.cloud.client.loadbalancer.Response<ServiceInstance> lbResponse = new DefaultResponse(
            instance);
    if (instance == null) {
        String message = "Load balancer does not contain an instance for the service " + serviceId;
        if (LOG.isWarnEnabled()) {
            LOG.warn(message);
        }
        supportedLifecycleProcessors.forEach(lifecycle -> lifecycle
            .onComplete(new CompletionContext<ResponseData, ServiceInstance, RequestDataContext>(
                    CompletionContext.Status.DISCARD, lbRequest, lbResponse)));
        return Response.builder()
            .request(request)
            .status(HttpStatus.SERVICE_UNAVAILABLE.value())
            .body(message, StandardCharsets.UTF_8)
            .build();
    }
    String reconstructedUrl = loadBalancerClient.reconstructURI(instance, originalUri).toString();
    Request newRequest = buildRequest(request, reconstructedUrl, instance);
    return executeWithLoadBalancerLifecycleProcessing(delegate, options, newRequest, lbRequest, lbResponse,
            supportedLifecycleProcessors);
}
```

第 2 行 `originalUri` 的值为 `http://example-user/user/1`，因此第 3 行获取到的 `serviceId` 为 `example-user`，这个值就是 `@FeignClient` 注解 `name` 属性的值。在第一次调用第 10 行的 `loadBalancerClientFactory.getInstances` 方法时会创建特定于每个 Feign 客户端的 Bean 容器，在刷新容器时会创建自定义的 `CanaryRoundRobinLoadBalancer` 对象。第 13 行的 `loadBalancerClient.choose` 调用会调用创建一个 `DiscoveryClientServiceInstanceListSupplier` 对象

```java
public DiscoveryClientServiceInstanceListSupplier(DiscoveryClient delegate, Environment environment) {
    this.serviceId = environment.getProperty(PROPERTY_NAME);
    resolveTimeout(environment);
    this.serviceInstances = Flux.defer(() -> Mono.fromCallable(() -> delegate.getInstances(serviceId)))
        .timeout(timeout, Flux.defer(() -> {
            logTimeout();
            return Flux.just(new ArrayList<>());
        }), Schedulers.boundedElastic())
        .onErrorResume(error -> {
            logException(error);
            return Flux.just(new ArrayList<>());
        });
}
```

第 2 行获取的 `serviceId` 为 `example-user`，第 4 行会调用 `CompositeDiscoveryClient` 的 `getInstances` 获取 `ServiceInstance`

```java
public List<ServiceInstance> getInstances(String serviceId) {
    if (this.discoveryClients != null) {
        for (DiscoveryClient discoveryClient : this.discoveryClients) {
            List<ServiceInstance> instances = discoveryClient.getInstances(serviceId);
            if (instances != null && !instances.isEmpty()) {
                return instances;
            }
        }
    }
    return Collections.emptyList();
}
```

第 4 行会调用 `EurekaDiscoveryClient` 的 `getInstances` 获取 `ServiceInstance`

```java
public List<ServiceInstance> getInstances(String serviceId) {
    List<InstanceInfo> infos = this.eurekaClient.getInstancesByVipAddress(serviceId, false);
    List<ServiceInstance> instances = new ArrayList<>();
    for (InstanceInfo info : infos) {
        instances.add(new EurekaServiceInstance(info));
    }
    return instances;
}
```

第 2 行将 `serviceId`，即 `example-user`，当作 `vipAddress` 从服务注册中心获取 `InstanceInfo`，而 `vipAddress` 在服务注册时正是服务提供方的 `spring.application.name` 属性的值，它是小写形式，因此在 Feign 客户端自定义负载均衡时名称需要小写。

完~
