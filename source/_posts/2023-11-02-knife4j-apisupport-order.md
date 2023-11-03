---
title: 一种解决 Knife4j 的 @ApiSupport 不生效的方法
date: 2023-11-02 21:16:46
tags:
---

## 问题描述

在我们的一个项目中使用了 Knife4j 来显示 Swagger 的文档，具体的依赖如下

```xml
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-openapi3-jakarta-spring-boot-starter</artifactId>
    <version>4.3.0</version>
</dependency>
```

同时我们有两个 Controller，它们分别是

```java
@RestController
@RequestMapping("/user")
@Tag(name = "用户管理")
public class UserController {
    // 省略了...
}

@RestController
@RequestMapping("/role")
@Tag(name = "角色管理")
public class RoleController {
    // 省略了...
}
```

默认情况下，在 Knife4j 的页面中“角色管理”显示在了“用户管理”的前面。我们期望“用户管理”显示在“角色管理”的前面。为了达到这个目的我们使用了 Knife4j 提供的注解 `@ApiSupport`，修改后的代码如下所示

```java
@RestController
@RequestMapping("/user")
@Tag(name = "用户管理")
@ApiSupport(order = 1)
public class UserController {
    // 省略了...
}

@RestController
@RequestMapping("/role")
@Tag(name = "角色管理")
@ApiSupport(order = 2)
public class RoleController {
    // 省略了...
}
```

但是并没有什么效果，“用户管理”并没有显示在“角色管理”的前面，即 `@ApiSupport` 注解没有生效。

<!-- more -->

## 原因分析

因为在前述依赖的 Knife4j 版本下，它依赖了 [springdoc-openapi](https://springdoc.org/)。springdoc-openapi 是在 `OpenAPIService` 的 `addTags` 方法解析 `io.swagger.v3.oas.annotations.tags.Tag` 注解构建 `io.swagger.v3.oas.models.tags.Tag` 对象

```java
private void addTags(List<Tag> sourceTags, Set<io.swagger.v3.oas.models.tags.Tag> tags, Locale locale) {
    Optional<Set<io.swagger.v3.oas.models.tags.Tag>> optionalTagSet = AnnotationsUtils
            .getTags(sourceTags.toArray(new Tag[0]), true);
    optionalTagSet.ifPresent(tagsSet -> {
        tagsSet.forEach(tag -> {
            tag.name(propertyResolverUtils.resolve(tag.getName(), locale));
            tag.description(propertyResolverUtils.resolve(tag.getDescription(), locale));
            if (tags.stream().noneMatch(t -> t.getName().equals(tag.getName())))
                tags.add(tag);
        });
    });
}
```

注意到这个方法将解析 `@Tag` 注解并构建 `Tag` 类型的功能的功能委托给了 `AnnotationsUtils` 类的 `getTags` 方法

```java
public static Optional<Set<Tag>> getTags(io.swagger.v3.oas.annotations.tags.Tag[] tags, boolean skipOnlyName) {
    if (tags == null) {
        return Optional.empty();
    }
    Set<Tag> tagsList = new LinkedHashSet<>();
    for (io.swagger.v3.oas.annotations.tags.Tag tag : tags) {
        if (StringUtils.isBlank(tag.name())) {
            continue;
        }
        if (skipOnlyName &&
                StringUtils.isBlank(tag.description()) &&
                StringUtils.isBlank(tag.externalDocs().description()) &&
                StringUtils.isBlank(tag.externalDocs().url())) {
            continue;
        }
        Tag tagObject = new Tag();
        if (StringUtils.isNotBlank(tag.description())) {
            tagObject.setDescription(tag.description());
        }
        tagObject.setName(tag.name());
        getExternalDocumentation(tag.externalDocs()).ifPresent(tagObject::setExternalDocs);
        if (tag.extensions().length > 0) {
            Map<String, Object> extensions = AnnotationsUtils.getExtensions(tag.extensions());
            if (extensions != null) {
                extensions.forEach(tagObject::addExtension);
            }
        }
        tagsList.add(tagObject);
    }
    if (tagsList.isEmpty()) {
        return Optional.empty();
    }
    return Optional.of(tagsList);
}
```

注意到第 7 行到第 15 行的两个条件判断，它们的意思是如果 `@Tag` 注解的 `name` 属性为空则不构建 `Tag` 对象，如果调用方式传入的 `skipOnlyName` 为 `true`，那么当只有 `name` 属性有值时也不构建 `Tag` 对象。前面的例子完全符合这两个条件，因此最终没有构建 `Tag` 对象。这一结果会导致最终创建的 `OpenAPI` 对象的 `tags` 属性为空（页面能正常显示两个标签那是另一个问题）。

那我们为 `@ApiSupport` 注解的 `description` 属性也赋上值看看，修改后的代码如下所示

```java
@RestController
@RequestMapping("/user")
@Tag(name = "用户管理", description = "用户管理")
@ApiSupport(order = 1)
public class UserController {
    // 省略了...
}

@RestController
@RequestMapping("/role")
@Tag(name = "角色管理", description = "角色管理")
@ApiSupport(order = 2)
public class RoleController {
    // 省略了...
}
```

非常遗憾的是这并没有达到我们想要的效果，肯定还有其他遗漏的地方。我们继续分析。

经过上面的修改我们已经能够正常构建 `Tag` 对象，即 `OpenAPI` 对象的 `tags` 属性有值了，通过 `/v3/api-docs` 接口获取到了 Swagger 文档也正常的返回了 `tags` 对象

```json
{
    // 省略了...
    "tags": [
        {
            "name": "角色管理",
            "description": "角色管理"
        },
        {
            "name": "用户管理",
            "description": "用户管理"
        }
    ],
    // 省略了...
}
```

我们注意到它没有 Knife4j 用来排序用的 `x-order` 属性。我们得想个办法把 `x-order` 属性加上，并且它的值就是 `@ApiSupport` 注解的 `order` 属性的值。

为了达到这个目的我们需要实现我们自己的 `OpenAPIService`，在构建好 `Tag` 对象后把 `x-order` 属性加在它的扩展属性 `extensions` 里，即我们要继承 `OpenAPIService` 类，并重写它的 `buildTags` 方法

```java
import com.github.xiaoymin.knife4j.annotations.ApiSupport;
import com.github.xiaoymin.knife4j.core.conf.ExtensionsConstants;
import io.swagger.v3.core.util.AnnotationsUtils;
import io.swagger.v3.oas.annotations.tags.Tag;
import io.swagger.v3.oas.annotations.tags.Tags;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.Operation;
import org.springdoc.core.customizers.OpenApiBuilderCustomizer;
import org.springdoc.core.customizers.ServerBaseUrlCustomizer;
import org.springdoc.core.properties.SpringDocConfigProperties;
import org.springdoc.core.providers.JavadocProvider;
import org.springdoc.core.service.OpenAPIService;
import org.springdoc.core.service.SecurityService;
import org.springdoc.core.utils.PropertyResolverUtils;
import org.springframework.core.annotation.AnnotatedElementUtils;
import org.springframework.core.annotation.AnnotationUtils;
import org.springframework.util.CollectionUtils;
import org.springframework.web.method.HandlerMethod;

import java.util.ArrayList;
import java.util.HashSet;
import java.util.List;
import java.util.Locale;
import java.util.Optional;
import java.util.Set;
import java.util.stream.Collectors;
import java.util.stream.Stream;

public class CustomOpenAPIService extends OpenAPIService {
    private final PropertyResolverUtils propertyResolverUtils;

    public CustomOpenAPIService(Optional<OpenAPI> openAPI, SecurityService securityParser, SpringDocConfigProperties springDocConfigProperties, PropertyResolverUtils propertyResolverUtils, Optional<List<OpenApiBuilderCustomizer>> openApiBuilderCustomizers, Optional<List<ServerBaseUrlCustomizer>> serverBaseUrlCustomizers, Optional<JavadocProvider> javadocProvider) {
        super(openAPI, securityParser, springDocConfigProperties, propertyResolverUtils, openApiBuilderCustomizers, serverBaseUrlCustomizers, javadocProvider);
        this.propertyResolverUtils = propertyResolverUtils;
    }

    @Override
    public Operation buildTags(HandlerMethod handlerMethod, Operation operation, OpenAPI openAPI, Locale locale) {
        super.buildTags(handlerMethod, operation, openAPI, locale);

        // 下面实现将 x-order 属性添加到 Tag 对象的 extensions 属性里

        Set<io.swagger.v3.oas.models.tags.Tag> tags = new HashSet<>();
        buildTags(handlerMethod.getBeanType(), tags, locale);

        ApiSupport apiSupport = AnnotationUtils.findAnnotation(handlerMethod.getBeanType(), ApiSupport.class);

        if (!CollectionUtils.isEmpty(openAPI.getTags()) && !CollectionUtils.isEmpty(tags) && apiSupport != null) {
            for (io.swagger.v3.oas.models.tags.Tag tag : tags) {
                List<io.swagger.v3.oas.models.tags.Tag> tagetTags = openAPI.getTags().stream().filter(e -> e.equals(tag)).toList();
                for (io.swagger.v3.oas.models.tags.Tag tagetTag : tagetTags) {
                    tagetTag.addExtension(ExtensionsConstants.EXTENSION_ORDER, apiSupport.order());
                }
            }
        }

        return operation;
    }

    // 参考 OpenAPIService 里的相关实现
    private void buildTags(Class<?> beanType, Set<io.swagger.v3.oas.models.tags.Tag> tags, Locale locale) {
        Set<Tags> tagsSet = AnnotatedElementUtils.findAllMergedAnnotations(beanType, Tags.class);
        Set<Tag> classTags = tagsSet.stream().flatMap(x -> Stream.of(x.value())).collect(Collectors.toSet());
        classTags.addAll(AnnotatedElementUtils.findAllMergedAnnotations(beanType, Tag.class));
        if (!CollectionUtils.isEmpty(classTags)) {
            List<Tag> allTags = new ArrayList<>(classTags);
            addTags(allTags, tags, locale);
        }
    }

    // 完全拷贝至父类的同名方法
    private void addTags(List<Tag> sourceTags, Set<io.swagger.v3.oas.models.tags.Tag> tags, Locale locale) {
        Optional<Set<io.swagger.v3.oas.models.tags.Tag>> optionalTagSet = AnnotationsUtils.getTags(sourceTags.toArray(new Tag[0]), true);
        optionalTagSet.ifPresent(tagsSet -> {
            tagsSet.forEach(tag -> {
                tag.name(propertyResolverUtils.resolve(tag.getName(), locale));
                tag.description(propertyResolverUtils.resolve(tag.getDescription(), locale));
                if (tags.stream().noneMatch(t -> t.getName().equals(tag.getName()))) {
                    tags.add(tag);
                }
            });
        });
    }
}
```

然后我们实现一个配置类，把它加入 Spring 的容器中

```java
import io.swagger.v3.oas.models.OpenAPI;
import org.springdoc.core.customizers.OpenApiBuilderCustomizer;
import org.springdoc.core.customizers.ServerBaseUrlCustomizer;
import org.springdoc.core.properties.SpringDocConfigProperties;
import org.springdoc.core.providers.JavadocProvider;
import org.springdoc.core.service.SecurityService;
import org.springdoc.core.utils.PropertyResolverUtils;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;
import java.util.Optional;

@Configuration
public class CustomSwaggerConfiguration {
    @Bean
    public CustomOpenAPIService customOpenAPIService(Optional<OpenAPI> openAPI,
                                                     SecurityService securityParser,
                                                     SpringDocConfigProperties springDocConfigProperties,
                                                     PropertyResolverUtils propertyResolverUtils,
                                                     Optional<List<OpenApiBuilderCustomizer>> openApiBuilderCustomizers,
                                                     Optional<List<ServerBaseUrlCustomizer>> serverBaseUrlCustomizers,
                                                     Optional<JavadocProvider> javadocProvider) {
        return new CustomOpenAPIService(openAPI, securityParser, springDocConfigProperties, propertyResolverUtils, openApiBuilderCustomizers, serverBaseUrlCustomizers, javadocProvider);
    }
}
```

现在调用 `/v3/api-docs` 接口，返回的 Swagger 文档数据中已经有 `x-order` 属性的值了

```json
{
    // 省略了...
    "tags": [
        {
            "name": "角色管理",
            "description": "角色管理",
            "x-order": 2
        },
        {
            "name": "用户管理",
            "description": "用户管理",
            "x-order": 1
        }
    ],
    // 省略了...
}
```

Knife4j 显示的顺序也符合我们的期望了。

## 解决问题

经过上面的分析后，解决问题主要在两个方面

1. `@Tag` 注解除了要给 `name` 属性赋值外，至少还应该给 `description`、`externalDocs.description`、`externalDocs.url` 之一进行赋值
2. 重写 `OpenAPIService` 类的 `buildTags` 方法，把 `x-order` 属性添加到 `Tag` 类的扩展属性 `extensions` 中
