---
title: 在 Spring Boot 应用中动态创建/销毁 MCP 服务器
date: 2025-11-15 21:10:18
updated: 2025-11-15 21:10:18
tags: [Spring AI, MCP]
---

在 Spring Boot 应用中使用 `org.springframework.ai:spring-ai-starter-mcp-server-webflux:1.1.0` 可以创建一个 MCP 服务器，如果想在一个应用创建多个 MCP 服务器，每个服务器使用不同的配置，比如不同的协议，该怎么实现呢？通过阅读 MCP 相关的源代码发现创建一个 MCP 服务器会向 Spring 容器注册 3 个对象（以异步的 Streamable HTTP 为例）：
`WebFluxStreamableServerTransportProvider`、`RouterFunction`、`McpAsyncServer`，只要能在应用启动中或启动后从配置文件或数据库读取配置然后创建这些对象，就可以实现动态创建/销毁 MCP 服务器。

<!-- more -->

## 配置文件

这里简单地使用配置文件来配置 MCP 服务器，因此需要定义一个配置属性的承载对象

```java
import org.springframework.ai.mcp.server.common.autoconfigure.properties.McpServerProperties;
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

@ConfigurationProperties(DynamicMcpServerProperties.CONFIG_PREFIX)
public class DynamicMcpServerProperties {
    public static final String CONFIG_PREFIX = "spring.ai.mcp.server.dynamic";

    private final Map<String, ServerParameters> servers = new HashMap<>();

    public Map<String, ServerParameters> getServers() {
        return this.servers;
    }

    public static class ServerParameters {
        private boolean enabled = true;
        private String version = "1.0.0";
        private McpServerProperties.ServerProtocol protocol;
        private Set<String> toolObjectNames = new HashSet<>();

        public boolean isEnabled() {
            return enabled;
        }

        public void setEnabled(boolean enabled) {
            this.enabled = enabled;
        }

        public String getVersion() {
            return version;
        }

        public void setVersion(String version) {
            this.version = version;
        }

        public McpServerProperties.ServerProtocol getProtocol() {
            return protocol;
        }

        public void setProtocol(McpServerProperties.ServerProtocol protocol) {
            this.protocol = protocol;
        }

        public Set<String> getToolObjectNames() {
            return toolObjectNames;
        }

        public void setToolObjectNames(Set<String> toolObjectNames) {
            this.toolObjectNames = toolObjectNames;
        }
    }
}
```

`servers` 配置的 *KEY* 为 MCP 服务器的名字，这个名字会作为访问端点的一部分；`tool-object-names` 是包含 `@Tool` 注解的 Bean 的名字。在这种方式下配置文件的示例如下所示

```yaml
server:
  port: 9093
spring:
  application:
    name: dynamic-mcp-server
  ai:
    mcp:
      server:
        dynamic:
          servers:
            mcp-server1:
              enabled: true
              protocol: SSE
              version: 1.0.0
              tool-object-names:
                - orderService
            mcp-server2:
              enabled: true
              protocol: STREAMABLE
              version: 1.0.0
              tool-object-names:
                - weatherService
```

在上述配置示例下，MCP 服务器的访问端点分别为 `http://localhost:9093/mcp-server1/sse` 和 `http://localhost:9093/mcp-server2/mcp`。

## 辅助工具

为了方便向 Spring 容器注册 Bean 和从 Spring 容器中获取 Bean，需要定义一个工具类

```java
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ConfigurableApplicationContext;

public final class SpringBeanUtils {
    private static final SpringBeanUtils INSTANCE = new SpringBeanUtils();

    private ApplicationContext applicationContext;

    private SpringBeanUtils() {
    }

    public static SpringBeanUtils getInstance() {
        return INSTANCE;
    }

    @SuppressWarnings("all")
    public <T> T getBean(final String beanName) {
        return (T) applicationContext.getBean(beanName);
    }

    public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
        return applicationContext.getBean(name, requiredType);
    }

    public void registerSingleton(String beanName, Object singletonObject) {
        DefaultListableBeanFactory beanFactory = getBeanFactory();
        beanFactory.registerSingleton(beanName, singletonObject);
    }

    public void destroySingleton(String beanName) {
        DefaultListableBeanFactory beanFactory = getBeanFactory();
        beanFactory.destroySingleton(beanName);
    }

    public boolean containsSingleton(String beanName) {
        DefaultListableBeanFactory beanFactory = getBeanFactory();
        return beanFactory.containsSingleton(beanName);
    }

    private DefaultListableBeanFactory getBeanFactory() {
        ConfigurableApplicationContext configurableApplicationContext = (ConfigurableApplicationContext) applicationContext;
        return (DefaultListableBeanFactory) configurableApplicationContext.getBeanFactory();
    }

    public void setApplicationContext(final ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    public ApplicationContext getApplicationContext() {
        return applicationContext;
    }
}
```

## 动态路由

Spring Boot 应用通过 `RouterFunction` 暴露 MCP 服务器的访问端点，然而 Spring Boot 的路由是静态的，在应用启动时就要定义好，在应用运行过程中没有办法动态地增加/删除路由，因此无法实现动态创建/销毁 MCP 服务器。要实现动态创建/销毁 MCP 服务器需要自己来管理路由，在这个管理器中存储所有 MCP 服务器的路由信息，即 `routerFunction` 变量

```java
import org.springframework.web.reactive.function.server.RouterFunction;
import reactor.core.publisher.Mono;

public class DynamicRouterFunctionManager {
    private static final RouterFunction<?> EMPTY_ROUTER_FUNCTION = request -> Mono.empty();

    private RouterFunction<?> routerFunction;

    public DynamicRouterFunctionManager() {
        this.routerFunction = EMPTY_ROUTER_FUNCTION;
    }

    public synchronized void refreshRouterFunction(RouterFunction<?> routerFunction) {
        this.routerFunction = routerFunction != null ? routerFunction : EMPTY_ROUTER_FUNCTION;
    }

    public synchronized RouterFunction<?> gettingRouterFunction() {
        return this.routerFunction;
    }
}
```

有了路由管理器后就可以向 Spring 容器中注册一个顶级 `RouterFunction`，它的 `route` 方法从路由管理器中获取路由，然后返回与请求匹配的 `HandlerFunction`

```java
@Bean
@SuppressWarnings("unchecked")
public RouterFunction<?> dynamicMcpRouterFunction(DynamicRouterFunctionManager dynamicRouterFunctionManager) {
    return request -> dynamicRouterFunctionManager.gettingRouterFunction()
            .route(request)
            .map(handler -> (HandlerFunction<ServerResponse>) handler);
}
```

## 动态服务

动态创建 MCP 服务器的时机为应用启动完成后，因此监听 Spring 容器的 `ApplicationReadyEvent`，此时会完成 3 件事情：1、根据配置的协议类型创建对应的 `TransportProvider` 和 `McpServer`；2、缓存 MCP 服务器的配置信息和 `RouterFunction`；3、使用当前的 `RouterFunction` 重建路由。

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import io.modelcontextprotocol.json.jackson.JacksonMcpJsonMapper;
import io.modelcontextprotocol.server.McpAsyncServer;
import io.modelcontextprotocol.server.McpServer;
import io.modelcontextprotocol.server.transport.WebFluxSseServerTransportProvider;
import io.modelcontextprotocol.server.transport.WebFluxStreamableServerTransportProvider;
import io.modelcontextprotocol.spec.McpSchema;
import org.springframework.ai.mcp.McpToolUtils;
import org.springframework.ai.mcp.server.common.autoconfigure.properties.McpServerProperties;
import org.springframework.ai.support.ToolCallbacks;
import org.springframework.ai.tool.ToolCallback;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.event.EventListener;
import org.springframework.web.reactive.function.server.RouterFunction;

import java.time.Duration;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class DynamicMcpServerManager implements DisposableBean {
    private static final String BEAN_NAME_PREFIX = DynamicMcpServerManager.class.getName() + ".";
    private static final String TRANSPORT_PROVIDER_BEAN_NAME_PREFIX = BEAN_NAME_PREFIX + "transportProvider.";
    private static final String MCP_SERVER_BEAN_NAME_PREFIX = BEAN_NAME_PREFIX + "mcpServer.";

    private static final Map<String, DynamicMcpServerProperties.ServerParameters> ACTIVE_MCP_SERVERS = new ConcurrentHashMap<>();
    private static final Map<String, RouterFunction<?>> ACTIVE_ROUTER_FUNCTIONS = new ConcurrentHashMap<>();

    private final ObjectMapper objectMapper;
    private final DynamicMcpServerProperties dynamicMcpServerProperties;
    private final DynamicRouterFunctionManager dynamicRouterFunctionManager;

    public DynamicMcpServerManager(ObjectMapper objectMapper, DynamicMcpServerProperties dynamicMcpServerProperties, DynamicRouterFunctionManager dynamicRouterFunctionManager) {
        this.objectMapper = objectMapper;
        this.dynamicMcpServerProperties = dynamicMcpServerProperties;
        this.dynamicRouterFunctionManager = dynamicRouterFunctionManager;
    }

    @Override
    public void destroy() throws Exception {
        stopMcpServer();
    }

    @EventListener(value = {ApplicationReadyEvent.class})
    public void onApplicationReady(ApplicationReadyEvent event) {
        startMcpServer();
    }

    private void startMcpServer() {
        for (Map.Entry<String, DynamicMcpServerProperties.ServerParameters> entry : this.dynamicMcpServerProperties.getServers().entrySet()) {
            String serverName = entry.getKey();
            DynamicMcpServerProperties.ServerParameters serverParameters = entry.getValue();
            if (!serverParameters.isEnabled()) {
                continue;
            }
            startMcpServer(serverName, serverParameters);
            ACTIVE_MCP_SERVERS.put(serverName, serverParameters);
        }
    }

    private void startMcpServer(String serverName, DynamicMcpServerProperties.ServerParameters serverParameters) {
        List<Object> toolObjects = new ArrayList<>();
        for (String toolObjectName : serverParameters.getToolObjectNames()) {
            toolObjects.add(SpringBeanUtils.getInstance().getBean(toolObjectName));
        }
        ToolCallback[] toolCallbacks = ToolCallbacks.from(toolObjects.toArray());

        if (serverParameters.getProtocol() == McpServerProperties.ServerProtocol.SSE) {
            String messageEndpoint = "/" + serverName + "/mcp/message";
            String sseEndpoint = "/" + serverName + "/sse";
            WebFluxSseServerTransportProvider transportProvider = WebFluxSseServerTransportProvider.builder()
                    .jsonMapper(new JacksonMcpJsonMapper(this.objectMapper))
                    .messageEndpoint(messageEndpoint)
                    .sseEndpoint(sseEndpoint)
                    .keepAliveInterval(Duration.ofHours(24))
                    .build();
            SpringBeanUtils.getInstance().registerSingleton(TRANSPORT_PROVIDER_BEAN_NAME_PREFIX + serverName, transportProvider);

            McpAsyncServer mcpServer = McpServer.async(transportProvider)
                    .serverInfo(serverName, serverParameters.getVersion())
                    .capabilities(McpSchema.ServerCapabilities.builder()
                            .tools(true)
                            .logging()
                            .build())
                    .tools(McpToolUtils.toAsyncToolSpecifications(toolCallbacks))
                    .build();
            SpringBeanUtils.getInstance().registerSingleton(MCP_SERVER_BEAN_NAME_PREFIX + serverName, mcpServer);

            RouterFunction<?> routerFunction = transportProvider.getRouterFunction();
            ACTIVE_ROUTER_FUNCTIONS.put(serverName, routerFunction);
            rebuildRouterFunction();
        }

        if (serverParameters.getProtocol() == McpServerProperties.ServerProtocol.STREAMABLE) {
            String messageEndpoint = "/" + serverName + "/mcp";
            WebFluxStreamableServerTransportProvider transportProvider = WebFluxStreamableServerTransportProvider.builder()
                    .jsonMapper(new JacksonMcpJsonMapper(this.objectMapper))
                    .messageEndpoint(messageEndpoint)
                    .keepAliveInterval(Duration.ofHours(24))
                    .disallowDelete(false)
                    .build();
            SpringBeanUtils.getInstance().registerSingleton(TRANSPORT_PROVIDER_BEAN_NAME_PREFIX + serverName, transportProvider);

            McpAsyncServer mcpServer = McpServer.async(transportProvider)
                    .serverInfo(serverName, serverParameters.getVersion())
                    .capabilities(McpSchema.ServerCapabilities.builder()
                            .tools(true)
                            .logging()
                            .build())
                    .tools(McpToolUtils.toAsyncToolSpecifications(toolCallbacks))
                    .build();
            SpringBeanUtils.getInstance().registerSingleton(MCP_SERVER_BEAN_NAME_PREFIX + serverName, mcpServer);

            RouterFunction<?> routerFunction = transportProvider.getRouterFunction();
            ACTIVE_ROUTER_FUNCTIONS.put(serverName, routerFunction);
            rebuildRouterFunction();
        }
    }

    private void rebuildRouterFunction() {
        RouterFunction<?> combined = ACTIVE_ROUTER_FUNCTIONS.values().stream().reduce(RouterFunction::andOther).orElse(null);
        dynamicRouterFunctionManager.refreshRouterFunction(combined);
    }

    private void stopMcpServer() {
        ACTIVE_MCP_SERVERS.forEach(this::stopMcpServer);
        ACTIVE_MCP_SERVERS.clear();
    }

    private void stopMcpServer(String serverName, DynamicMcpServerProperties.ServerParameters serverParameters) {
        ACTIVE_MCP_SERVERS.remove(serverName);
        rebuildRouterFunction();

        if (SpringBeanUtils.getInstance().containsSingleton(MCP_SERVER_BEAN_NAME_PREFIX + serverName)) {
            SpringBeanUtils.getInstance().destroySingleton(MCP_SERVER_BEAN_NAME_PREFIX + serverName);
        }

        if (SpringBeanUtils.getInstance().containsSingleton(TRANSPORT_PROVIDER_BEAN_NAME_PREFIX + serverName)) {
            if (ACTIVE_MCP_SERVERS.get(serverName).getProtocol() == McpServerProperties.ServerProtocol.SSE) {
                WebFluxSseServerTransportProvider transportProvider = SpringBeanUtils.getInstance().getBean(TRANSPORT_PROVIDER_BEAN_NAME_PREFIX + serverName, WebFluxSseServerTransportProvider.class);
                transportProvider.close();
            }
            if (ACTIVE_MCP_SERVERS.get(serverName).getProtocol() == McpServerProperties.ServerProtocol.STREAMABLE) {
                WebFluxStreamableServerTransportProvider transportProvider = SpringBeanUtils.getInstance().getBean(TRANSPORT_PROVIDER_BEAN_NAME_PREFIX + serverName, WebFluxStreamableServerTransportProvider.class);
                transportProvider.close();
            }
            SpringBeanUtils.getInstance().destroySingleton(TRANSPORT_PROVIDER_BEAN_NAME_PREFIX + serverName);
        }
    }
}
```

另外，现在与 MCP 相关的对象都由我们自己管理，因此也提供了在容器销毁时的销毁方法 `stopMcpServer`。这个功能在当前的实现中作用并不明显，当使用数据库作为存储配置的源时，这个功能就会有作用，此时可能会有一个管理后台来维护配置，进行 MCP 服务器的启/停，工具的增/删。比如在管理后台维护配置后向 Redis 发送一条广播消息，在 MCP 服务器中监听到这个消息后，就可以根据消息中的内容来启动/停止 MCP 服务器。下面是一个示例

```java
@Bean
public RedisMessageListenerContainer messageListenerContainer(RedisConnectionFactory connectionFactory, DynamicMcpServerManager dynamicMcpServerManager) {
    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(connectionFactory);
    container.addMessageListener(
            (message, pattern) -> {
                String mcpServerName = new String(message.getBody(), StandardCharsets.UTF_8);
                dynamicMcpServerManager.startMcpServer(mcpServerName);
            },
            new ChannelTopic("spring:ai:mcp:server:dynamic:server:start")
    );
    container.addMessageListener(
            (message, pattern) -> {
                String mcpServerName = new String(message.getBody(), StandardCharsets.UTF_8);
                dynamicMcpServerManager.stopMcpServer(mcpServerName);
            },
            new ChannelTopic("spring:ai:mcp:server:dynamic:server:stop")
    );
    container.addMessageListener(
            (message, pattern) -> {
                String mcpServerName = new String(message.getBody(), StandardCharsets.UTF_8);
                dynamicMcpServerManager.syncMcpTool(mcpServerName);
            },
            new ChannelTopic("spring:ai:mcp:server:dynamic:tool:change")
    );
    return container;
}
```

在 `DynamicMcpServerManager` 中添加了 `startMcpServer`、`stopMcpServer` 和 `syncMcpTool` 方法，其中 `syncMcpTool` 方法只简单实现了添加工具功能，没有实现删除工具的功能，想要实现真正的工具同步功能还需要实现工具历史工具和最新工具之间的比较逻辑，这部分需要根据实际需求进行实现。

```java
public class DynamicMcpServerManager implements DisposableBean {
    // 省略了...

    public void startMcpServer(String serverName) {
        DynamicMcpServerProperties.ServerParameters serverParameters = this.dynamicMcpServerProperties.getServers().get(serverName);
        startMcpServer(serverName, serverParameters);
        ACTIVE_MCP_SERVERS.put(serverName, serverParameters);
    }

    public void stopMcpServer(String serverName) {
        DynamicMcpServerProperties.ServerParameters serverParameters = ACTIVE_MCP_SERVERS.get(serverName);
        stopMcpServer(serverName, serverParameters);
    }

    public void syncMcpTool(String serverName) {
        DynamicMcpServerProperties.ServerParameters serverParameters = this.dynamicMcpServerProperties.getServers().get(serverName);
        List<Object> toolObjects = new ArrayList<>();
        for (String toolObjectName : serverParameters.getToolObjectNames()) {
            toolObjects.add(SpringBeanUtils.getInstance().getBean(toolObjectName));
        }
        ToolCallback[] toolCallbacks = ToolCallbacks.from(toolObjects.toArray());
        McpAsyncServer mcpServer = SpringBeanUtils.getInstance().getBean(MCP_SERVER_BEAN_NAME_PREFIX + serverName, McpAsyncServer.class);
        for (ToolCallback toolCallback : toolCallbacks) {
            mcpServer.addTool(McpToolUtils.toAsyncToolSpecification(toolCallback)).block();
        }
    }

    // 省略了...
}
```

## 整合集成

现在需要的所有组件都已经实现，下面需要做的就是把它们集成起来

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.reactive.function.server.HandlerFunction;
import org.springframework.web.reactive.function.server.RouterFunction;
import org.springframework.web.reactive.function.server.ServerResponse;

@Configuration
public class DynamicMcpServerConfiguration implements ApplicationContextAware {
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        SpringBeanUtils.getInstance().setApplicationContext(applicationContext);
    }

    @Bean
    public DynamicMcpServerProperties dynamicMcpServerProperties() {
        return new DynamicMcpServerProperties();
    }

    @Bean
    public DynamicRouterFunctionManager dynamicRouterFunctionManager() {
        return new DynamicRouterFunctionManager();
    }

    @Bean
    @SuppressWarnings("unchecked")
    public RouterFunction<?> dynamicMcpRouterFunction(DynamicRouterFunctionManager dynamicRouterFunctionManager) {
        return request -> dynamicRouterFunctionManager.gettingRouterFunction()
                .route(request)
                .map(handler -> (HandlerFunction<ServerResponse>) handler);
    }

    @Bean
    public DynamicMcpServerManager dynamicMcpServerManager(@Qualifier("mcpServerObjectMapper") ObjectMapper objectMapper, DynamicMcpServerProperties dynamicMcpServerProperties, DynamicRouterFunctionManager dynamicRouterFunctionManager) {
        return new DynamicMcpServerManager(objectMapper, dynamicMcpServerProperties, dynamicRouterFunctionManager);
    }
}
```

至此，一个简易版本的动态创建/销毁 MCP 服务器的功能就实现好了，完整的代码示例在 [Github](https://github.com/acomma/examples/tree/main/example-mcp/mcp-server-dynamic)。

## 参考资料

1. [spring ai mcp server multi-instance solution](https://github.com/spring-projects/spring-ai/issues/3515)
2. [Dynamic Tool Updates in Spring AI's Model Context Protocol](https://spring.io/blog/2025/05/04/spring-ai-dynamic-tool-updates-with-mcp)
