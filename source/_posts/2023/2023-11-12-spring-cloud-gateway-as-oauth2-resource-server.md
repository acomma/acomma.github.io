---
title: 将 Spring Cloud Gateway 作为 OAuth2 的资源服务器
date: 2023-11-12 14:46:15
updated: 2023-11-12 14:46:15
tags: [OAuth2, Spring Cloud]
---

我们将要实现这样一个微服务系统，将 Spring Cloud Gateway 作为 OAuth2 的资源服务器，在 Gateway 实现集中的统一的鉴权功能，各个微服务之间的调用不再单独鉴权。系统的规划如下表所示

系统 | 端口 | 说明
--- | --- | ---
example-auth | 9090 | 认证与授权服务
example-eureka | 8761 | 服务注册中心
example-gateway | 8080 | 服务网关
example-user | 8081 | 用户服务
example-product | 8082 | 商品服务
example-order | 8083 | 订单服务
example-common | - | 公共模块

example-auth 是一个独立的服务，它不会向服务注册中心注册自己，其他的服务，包括 example-gateway，均需要向服务注册中心注册自己。

<!-- more -->

我们将创建一个名称为 `example-cloud` 的多模块项目，项目的 `pom.xml` 文件的内容如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.1.1</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>com.example</groupId>
    <artifactId>example-cloud</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>pom</packaging>

    <name>example-cloud</name>
    <description>example-cloud</description>

    <modules>
        <module>example-eureka</module>
        <module>example-gateway</module>
        <module>example-user</module>
        <module>example-product</module>
        <module>example-order</module>
        <module>example-common</module>
        <module>example-auth</module>
    </modules>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <spring-cloud.version>2022.0.4</spring-cloud.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

从这个文件不难发现我们使用的 Spring Boot 版本为 `3.1.1`，Spring Cloud 的版本为 `2022.0.4`。下面我们来逐步的完善各个模块。

## 认证与授权服务

`example-auth` 模块的内容和[前一篇文章](https://acomma.github.io/2023/11/05/spring-authorization-server-practice)差不多，它的 `pom.xml` 文件的内容如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>example-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>example-auth</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

不难发现它只有一个依赖 `spring-boot-starter-oauth2-authorization-server`。

### 配置文件

配置文件 `application.yml` 的内容如下所示

```yaml
server:
  port: 9090
spring:
  application:
    name: example-auth
```

我们只配置了端口和名称。

### Web 安全配置

Web 安全配置类 `WebSecurityConfig.java` 的代码如下所示

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.core.userdetails.User;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.provisioning.InMemoryUserDetailsManager;
import org.springframework.security.web.SecurityFilterChain;

@Configuration(proxyBeanMethods = false)
public class WebSecurityConfig {
    @Bean
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests((authorize) -> authorize.anyRequest().authenticated())
                // Form login handles the redirect to the login page from the
                // authorization server filter chain
                .formLogin(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails userDetails = User.withDefaultPasswordEncoder()
                .username("bob")
                .password("123456")
                .roles("USER")
                .build();
        return new InMemoryUserDetailsManager(userDetails);
    }
}
```

配置类的内容来自 [Spring Authorization Server Reference - Getting Started - Defining Required Components](https://docs.spring.io/spring-authorization-server/docs/1.1.3/reference/html/getting-started.html#defining-required-components)，只是把与 Web 安全相关的内容提取到了这个类。

### 授权服务器配置

```java
import com.nimbusds.jose.jwk.JWKSet;
import com.nimbusds.jose.jwk.RSAKey;
import com.nimbusds.jose.jwk.source.ImmutableJWKSet;
import com.nimbusds.jose.jwk.source.JWKSource;
import com.nimbusds.jose.proc.SecurityContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.http.MediaType;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.core.AuthorizationGrantType;
import org.springframework.security.oauth2.core.ClientAuthenticationMethod;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.server.authorization.client.InMemoryRegisteredClientRepository;
import org.springframework.security.oauth2.server.authorization.client.RegisteredClient;
import org.springframework.security.oauth2.server.authorization.client.RegisteredClientRepository;
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configuration.OAuth2AuthorizationServerConfiguration;
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configurers.OAuth2AuthorizationServerConfigurer;
import org.springframework.security.oauth2.server.authorization.settings.AuthorizationServerSettings;
import org.springframework.security.oauth2.server.authorization.settings.ClientSettings;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint;
import org.springframework.security.web.util.matcher.MediaTypeRequestMatcher;

import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.interfaces.RSAPrivateKey;
import java.security.interfaces.RSAPublicKey;
import java.util.UUID;

@Configuration(proxyBeanMethods = false)
public class OAuth2AuthorizationServerConfig {
    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);
        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
                // Enable OpenID Connect 1.0
                .oidc(Customizer.withDefaults());
        http
                // Redirect to the login page when not authenticated from the authorization endpoint
                .exceptionHandling((exceptions) -> exceptions.defaultAuthenticationEntryPointFor(
                        new LoginUrlAuthenticationEntryPoint("/login"),
                        new MediaTypeRequestMatcher(MediaType.TEXT_HTML))
                )
                // Accept access tokens for User Info and/or Client Registration
                .oauth2ResourceServer((resourceServer) -> resourceServer.jwt(Customizer.withDefaults()));
        return http.build();
    }

    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        RegisteredClient registeredClient = RegisteredClient.withId(UUID.randomUUID().toString())
                .clientId("example-client")
                .clientSecret("{noop}example-client-secret")
                .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
                .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
                .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
                .redirectUri("https://www.baidu.com")
                .scope("user")
                .scope("product")
                .scope("order")
                .clientSettings(ClientSettings.builder().requireAuthorizationConsent(true).build())
                .build();
        return new InMemoryRegisteredClientRepository(registeredClient);
    }

    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        KeyPair keyPair = generateRsaKey();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        RSAKey rsaKey = new RSAKey.Builder(publicKey)
                .privateKey(privateKey)
                .keyID(UUID.randomUUID().toString())
                .build();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return new ImmutableJWKSet<>(jwkSet);
    }

    private static KeyPair generateRsaKey() {
        KeyPair keyPair;
        try {
            KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
            keyPairGenerator.initialize(2048);
            keyPair = keyPairGenerator.generateKeyPair();
        } catch (Exception ex) {
            throw new IllegalStateException(ex);
        }
        return keyPair;
    }

    @Bean
    public JwtDecoder jwtDecoder(JWKSource<SecurityContext> jwkSource) {
        return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource);
    }

    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
                // 资源管理器需要在 spring.oauth2.resourceserver.jwt.issuer-uri 属性中配置这个地址
                .issuer("http://localhost:9090")
                .build();
    }
}
```

配置类的内容来自 [Spring Authorization Server Reference - Getting Started - Defining Required Components](https://docs.spring.io/spring-authorization-server/docs/1.1.3/reference/html/getting-started.html#defining-required-components)，只是把与授权服务器相关的内容提取到了这个类。

### 启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ExampleAuthApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExampleAuthApplication.class, args);
    }
}
```

## 服务注册中心

`example-eureka` 模块的内容基本都能在 [2. Service Discovery: Eureka Server](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/#spring-cloud-eureka-server) 找到，它的 `pom.xml` 文件的内容如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>example-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>example-eureka</artifactId>

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
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 配置文件

配置文件 `application.yml` 的内容如下所示

```yaml
server:
  port: 8761
eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

我们使用的是 [2.5. Standalone Mode](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/#spring-cloud-eureka-server-standalone-mode)。这里要注意 `eureka.client` 下面的属性的大小写，参考 [1.2. Registering with Eureka](https://docs.spring.io/spring-cloud-netflix/docs/current/reference/html/#registering-with-eureka) 下的警告信息

> The `defaultZone` property is case sensitive and requires camel case because the `serviceUrl` property is a `Map<String, String>`. Therefore, the `defaultZone` property does not follow the normal Spring Boot snake-case convention of `default-zone`.

还要注意的是配置 `server.port` 的值，`8761` 是 Eureka 的默认值。

### 启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class ExampleEurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExampleEurekaApplication.class, args);
    }
}
```

启动类是比较简单的，只需要注意别忘了在启动类上添加 `@EnableEurekaServer` 注解。

## 服务网关

`example-gateway` 模块的 `pom.xml` 文件的内容如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>example-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>example-gateway</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-gateway</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

### 配置文件

配置文件 `application.yml` 的内容如下所示

```yaml
server:
  port: 8080
spring:
  application:
    name: example-gateway
  cloud:
    gateway:
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:9090
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

Eureka 默认的会使用配置 `spring.application.name` 的值的大写作为服务的名称，比如 `EXAMPLE-USER`，将配置 `spring.cloud.gateway.discover.locator.lower-case-service-id` 的值设置为 `true`，可以在调用接口时使用小写形式，比如 `http://localhost:8080/example-user/user/1`。

配置 `spring.security.oauth2.resourceserver.jwt.issuer-uri` 的值是在 `example-auth` 模块的 `OAuth2AuthorizationServerConfig` 类的 `authorizationServerSettings` 方法中设置的值。

配置 `eureka.client.serverUrl.defaultZone` 的值需要注意它的端口 `8761`，同时要注意它的键是区分大小写的。

### 资源服务器配置

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.annotation.Order;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.web.server.ServerHttpSecurity;
import org.springframework.security.web.server.SecurityWebFilterChain;

@Configuration(proxyBeanMethods = false)
public class OAuth2ResourceServerConfig {
    @Bean
    @Order(1)
    public SecurityWebFilterChain resourceServerSecurityFilterChain(ServerHttpSecurity http) {
        http.authorizeExchange(exchanges -> exchanges
                .pathMatchers("/example-user/**").hasAuthority("SCOPE_user")
                .pathMatchers("/example-product/**").hasAuthority("SCOPE_product")
                .pathMatchers("/example-order/**").hasAuthority("SCOPE_order")
                .anyExchange().authenticated());
        http.oauth2ResourceServer(configurer -> configurer.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

我们分别为用户服务、商品服务、订单服务设置的了它们需要的权限。配置的路径前缀都是服务的名称，因为通过网关请求接口时请求地址的格式为 `http://{gateway-domain}:{gateway-port}/{service-name}/{specific-path}`。`SCOPE_` 是固定的部分，它后面的部分为在 `example-auth` 中配置的 `scope`。

### 启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ExampleGatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExampleGatewayApplication.class, args);
    }
}
```

## 业务服务

业务服务都是比较简单的 Spring Boot 工程。

### 公共模块

`example-common` 模块定义了其他业务服务需要的实体类，它的 `pom.xml` 文件的内容如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>example-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>example-common</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

#### 用户实体

```java
import lombok.Data;

@Data
public class User {
    private Integer id;

    private String name;
}
```

#### 商品实体

```java
import lombok.Data;

@Data
public class Product {
    private Integer id;

    private String name;
}
```

#### 订单实体

```java
import lombok.Data;

@Data
public class Order {
    private String userName;

    private String productName;
}
```

### 用户服务

`example-user` 模块的 `pom.xml` 文件的内容如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>example-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>example-user</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>example-common</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

#### 配置文件

配置文件 `application.yml` 的内容如下所示

```yaml
server:
  port: 8081
spring:
  application:
    name: example-user
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

#### 用户接口

```java
import com.example.common.entity.User;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/user")
public class UserController {
    @GetMapping("/{userId}")
    public User getUser(@PathVariable("userId") Integer id) {
        User user = new User();
        user.setId(id);
        user.setName("user-" + id);
        return user;
    }
}
```

#### 启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ExampleUserApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExampleUserApplication.class, args);
    }
}
```

### 商品服务

`example-product` 模块的 `pom.xml` 文件的内容如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>example-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>example-product</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>example-common</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

#### 配置文件

配置文件 `application.yml` 的内容如下所示

```yaml
server:
  port: 8082
spring:
  application:
    name: example-product
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

#### 商品接口

```java
import com.example.common.entity.Product;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/product")
public class ProductController {
    @GetMapping("/{productId}")
    public Product getProduct(@PathVariable("productId") Integer id) {
        Product product = new Product();
        product.setId(id);
        product.setName("product-" + id);
        return product;
    }
}
```

#### 启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ExampleProductApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExampleProductApplication.class, args);
    }
}
```

### 订单服务

`example-order` 模块的 `pom.xml` 文件的内容如下所示

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>com.example</groupId>
        <artifactId>example-cloud</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </parent>

    <artifactId>example-order</artifactId>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
        <dependency>
            <groupId>com.example</groupId>
            <artifactId>example-common</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

与其他两个服务不同的是订单服务多引入了一个 `spring-cloud-starter-openfeign` 依赖，我们将在订单服务里实现对其他连个服务的调用，以验证在我们的实现中内部服务之间的调用不需要鉴权。

#### 配置文件

配置文件 `application.yml` 的内容如下所示

```yaml
server:
  port: 8083
spring:
  application:
    name: example-order
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

#### 客户端

用户客户端

```java
import com.example.common.entity.User;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "example-user")
public interface UserClient {
    @GetMapping("/user/{userId}")
    User getUser(@PathVariable("userId") Integer userId);
}
```

商品客户端

```java
@FeignClient(name = "example-product")
public interface ProductClient {
    @GetMapping("/product/{productId}")
    Product getProduct(@PathVariable("productId") Integer productId);
}
```

#### 订单接口

```java
import com.example.common.entity.Order;
import com.example.common.entity.Product;
import com.example.common.entity.User;
import com.example.order.client.ProductClient;
import com.example.order.client.UserClient;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/order")
@RequiredArgsConstructor
public class OrderController {
    private final UserClient userClient;
    private final ProductClient productClient;

    @GetMapping("/add")
    public Order add(Integer userId, Integer productId) {
        User user = userClient.getUser(userId);
        Product product = productClient.getProduct(productId);

        Order order = new Order();
        order.setUserName(user.getName());
        order.setProductName(product.getName());

        return order;
    }
}
```

我们模拟了新增订单的功能，它需要确定是那个用户新增了那个商品的订单，因此通过各自的客户端获取相应的信息。

#### 启动类

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableFeignClients
public class ExampleOrderApplication {
    public static void main(String[] args) {
        SpringApplication.run(ExampleOrderApplication.class, args);
    }
}
```

这里需要注意启动类上的 `@EnableFeignClients` 注解。

## 验证测试

{% asset_img postman-get-new-access-token.png %}

我们只验证测试新增订单接口，其他两个接口是类似的。请求的接口地址和参数，获取 Access Token 的各项配置如上图所示。

## 业务服务获取授权用户信息

在 Gateway 解析 Access Token，从中获取授权用户的信息，通过请求头 `Header` 的方式传给业务服务。

业务服务之间相互调用时可以实现一个 OpenFeign 的拦截器，把请求头 `Header` 中的用户信息继续传递给其他业务服务。

## 授权服务作为内部服务

目前授权服务 `example-auth` 是作为一个独立的服务单独部署的，它没有加入 `example-eureka` 服务注册中心，无法通过网关对它进行访问。授权服务如果要访问用户信息的话无法通过访问用户服务 `example-user` 得到，它可能会和用户服务共用数据库等资源。如果将授权服务纳入服务注册中心，通过网关进行访问会出现什么问题呢？

当在浏览器访问 [http://localhost:8080/example-auth/oauth2/authorize?client_id=example-client&scope=product&state=965236&response_type=code&redirect_uri=https://www.baidu.com](http://localhost:8080/example-auth/oauth2/authorize?client_id=example-client&scope=product&state=965236&response_type=code&redirect_uri=https://www.baidu.com) 获取授权码时会跳转到 `example-auth` 的登录页面

{% asset_img sign-in-page-of-example-auth.png %}

当我们输入用户名和密码登录后会跳转到 `example-auth` 的错误页面

{% asset_img whitelabel-error-page-of-example-auth.png %}

可以参考参考资料 [[13]](https://blog.csdn.net/weixin_43829936/article/details/118250290) 的方式进行解决。但是在点击 `Sign in` 按钮后会得到 CSRF 结果

{% asset_img csrf-of-gateway.png %}

当我们在网关使用 `http.csrf(ServerHttpSecurity.CsrfSpec::disable);` 禁用 csrf 后又会得到一个空白页面。

这个问题的本质是 `example-auth` 模块的 `DefaultLoginPageGeneratingFilter` 类返回的默认的登录页面的内容为

```html
<form class="form-signin" method="post" action="/login">
    <h2 class="form-signin-heading">Please sign in</h2>
    <p>
        <label for="username" class="sr-only">Username</label>
        <input type="text" id="username" name="username" class="form-control" placeholder="Username" required="" autofocus="">
    </p>
    <p>
        <label for="password" class="sr-only">Password</label>
        <input type="password" id="password" name="password" class="form-control" placeholder="Password" required="">
    </p>
    <input name="_csrf" type="hidden" value="ROsIkqxGnIV02wiqz_3sVIVFVy-Gf3Z6_7ywk0WFs486MLO9IdJrp84jqrBZ72qSqtDYMeNwek7kGkRXnN6FoHy8he0JVoaL">
    <button class="btn btn-lg btn-primary btn-block" type="submit">Sign in</button>
</form>
```

在点击 `Sign in` 按钮是会调用 `http://localhost:8080/login` 提交表单到 `example-gateway` 模块，在网关这个请求不会被转发，而在网关它又是没有权限的，因此会返回 `403 Forbidden`。

## 参考资料

1. [Using Spring Cloud Gateway with OAuth 2.0 Patterns](https://www.baeldung.com/spring-cloud-gateway-oauth2)，中文翻译 [Spring Cloud Gateway 和 Oauth2](https://springdoc.cn/spring-cloud-gateway-oauth2/)
2. [Securing Services with Spring Cloud Gateway](https://spring.io/blog/2019/08/16/securing-services-with-spring-cloud-gateway)
3. [微服务权限终极解决方案，Spring Cloud Gateway + Oauth2 实现统一认证和鉴权！](https://cloud.tencent.com/developer/article/1661343)
4. [07 网关集成OAuth2.0实现统一认证鉴权](https://java-family.cn/#/OAuth2.0/07-Spring-Cloud-Gateway%E9%9B%86%E6%88%90OAuth2.0)
5. [09 Spring Cloud Gateway集成 RBAC 权限模型实现动态权限控制！](https://java-family.cn/#/OAuth2.0/09-Spring-Cloud-Gateway%E9%9B%86%E6%88%90RBAC%E6%9D%83%E9%99%90%E6%A8%A1%E5%9E%8B%E5%AE%9E%E7%8E%B0%E5%8A%A8%E6%80%81%E6%9D%83%E9%99%90%E6%8E%A7%E5%88%B6%EF%BC%81)
6. [服务之间调用还需要鉴权？](https://cloud.tencent.com/developer/article/1661115)
7. [OAuth 2.0 Patterns with Spring Cloud Gateway](https://developer.okta.com/blog/2020/08/14/spring-gateway-patterns)
8. [关于微服务内部服务认证](https://zhuanlan.zhihu.com/p/478559129)
9. [微服务架构下的统一身份认证和授权](https://www.jianshu.com/p/2571f6a4e192)
10. [微服务下前后端分离的统一认证授权服务，基于Spring Security OAuth2 + Spring Cloud Gateway实现单点登录](https://www.cnblogs.com/cjsblog/p/15669043.html)
11. [分布式系统下的认证与授权](https://zhuanlan.zhihu.com/p/447795433)
12. [Spring Cloud实战 | 第六篇：Spring Cloud Gateway + Spring Security OAuth2 + JWT实现微服务统一认证授权鉴权](https://www.cnblogs.com/haoxianrui/p/13719356.html)
13. [oauth2授权码模式遇到的坑，1.走网关无法返回授权码 2.refresh_token新token丢失用户信息](https://blog.csdn.net/weixin_43829936/article/details/118250290)
14. [oauth2 通过gateway请求授权码不能回调到return_uri](https://blog.csdn.net/hc1285653662/article/details/126633112)
15. [记录一下spring security+oauth2 指定登陆后跳转路径失败原因](https://blog.csdn.net/weixin_45744265/article/details/104860214)
16. [oauth2授权码登录踩坑积累一（登录页无法跳转到授权页面）](https://blog.csdn.net/liu379333171/article/details/108507131)
17. [Spring Security认证成功后回跳（解决前后端分离下OAuth2认证成功回跳）](https://blog.csdn.net/gangsijay888/article/details/81171647)
18. [security,Oauth2登录后302重定向Location参数错误，Oauth2授权码模式直接获取access_token](https://blog.csdn.net/weixin_44530390/article/details/107834746)
19. [我加了网关之后，自定义登录页面，登录成功跳转不回/oauth/authorize 这个页面了。直接访问授权服务是可以的](https://coding.imooc.com/learn/questiondetail/150379.html)
20. [Spring Authorization Server入门 (十二) 实现授权码模式使用前后端分离的登录页面](https://juejin.cn/post/7254096495184134181)
