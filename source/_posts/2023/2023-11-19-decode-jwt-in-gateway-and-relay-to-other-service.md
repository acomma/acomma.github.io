---
title: 在网关解析 JWT 并中继给其他服务
date: 2023-11-19 16:51:49
updated: 2023-11-19 16:51:49
tags: [JWT, Spring Cloud]
---

通过前面几篇文章我们已经实现了将网关作为资源服务器并在网关实现认证和授权，其他服务将是一个纯粹的微服务，那么我们将问一个问题其他微服务如何获取到已授权用户的用户信息？

<!-- more -->

## 自定义网关请求过滤器

我们需要在网关实现一个请求过滤器，在这个过滤器里解析访问令牌，然后把解析出来的用户信息加入请求头中

```java
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpHeaders;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.security.oauth2.jwt.ReactiveJwtDecoder;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

/**
 * 自定义请求过滤器。参考 <a href="https://cloud.tencent.com/developer/article/2264294">Spring Cloud Security配置JWT和OAuth2的集成实现授权管理（三）</a>实现。
 */
@Component
public class CustomRequestFilter implements GlobalFilter, Ordered {
    private final ReactiveJwtDecoder reactiveJwtDecoder;

    public CustomRequestFilter(ReactiveJwtDecoder reactiveJwtDecoder) {
        this.reactiveJwtDecoder = reactiveJwtDecoder;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String accessToken = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);

        // Authorization 请求头为空的不处理
        if (accessToken == null || accessToken.isBlank()) {
            return chain.filter(exchange);
        }
        // Authorization 请求头不是以 “Bearer ” 开头的不处理，比如 “Basic ”
        if (!accessToken.startsWith("Bearer ")) {
            return chain.filter(exchange);
        }

        // 去掉 “Bearer ” 前缀
        accessToken = accessToken.replaceFirst("Bearer ", "");

        return reactiveJwtDecoder.decode(accessToken)
                .flatMap(jwt -> {
                    // 放入请求头的可以是从 JWT 中获取的用户信息，这里只是简单的把 Subject 信息放进去
                    ServerHttpRequest newRequest = exchange.getRequest().mutate().header("X-User", jwt.getSubject()).build();
                    ServerWebExchange newExchange = exchange.mutate().request(newRequest).build();
                    return chain.filter(newExchange);
                })
                .onErrorResume(throwable -> chain.filter(exchange));
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

当网关将请求转发到其他服务时，其他服务能够从请求头中获取到授权用户的信息，比如

```java
@GetMapping("/{userId}")
public User getUser(@PathVariable("userId") Integer id, HttpServletRequest request) {
    System.out.println("授权用户信息：" + request.getHeader("X-User"));
    // 省略了...
}
```

## 自定义 Feign 请求拦截器

通过上面的过滤器虽然服务能够获取到授权用户信息，但是当这个服务有需要调用其他服务时，其他服务并不能获取到授权用户信息，此时我们需要通过 Feign 的请求拦截器将授权用户信息加入 Feign 的请求头中，从而实现授权用户信息的中继功能

```java
import feign.RequestInterceptor;
import feign.RequestTemplate;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

/**
 * 自定义 Feign 请求拦截器，实现授权用户信息中继功能。这个拦截器需要定义在每一个需要中继功能的服务中，可能定义在公共模块更好一点，这里偷个懒儿。
 */
@Component
public class CustomRequestInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        ServletRequestAttributes requestAttributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        if (requestAttributes != null) {
            HttpServletRequest request = requestAttributes.getRequest();
            // 在实践中这里可以是已授权的用户信息
            String header = request.getHeader("X-User");
            if (header != null) {
                template.header("X-User", header);
            }
        }
    }
}
```

## 参考资料

1. [Spring Cloud Security配置JWT和OAuth2的集成实现授权管理（三）](https://cloud.tencent.com/developer/article/2264294)
2. [Spring Cloud Security Token Relay‘s Principle](https://blog.csdn.net/xichenguan/article/details/94379369)
