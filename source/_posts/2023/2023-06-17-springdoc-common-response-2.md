---
title: SpringDoc 统一应答类型处理（二）
date: 2023-06-17 13:03:02
updated: 2023-06-17 13:03:02
tags:
---

## 问题描述

在上一篇文章中我们通过继承 `GenericResponseService` 类并重写 `build` 方法和实现 `OperationCustomizer` 接口的方式实现了 SpringDoc 的统一应答类型处理，但是实现的方式有点不够好，侵入 SpringDoc 的流程比较多，是否有更好的实现方式呢？

<!-- more -->

## 问题分析

继续阅读 `GenericResponseService` 类的 `build` 方法的实现，我们发现 SpringDoc 是从 `java.lang.reflect.Type` 类型的子类中提取 Schema 的，比如 `User` 的类型是 `User.class`，而这个类型是通过下面的方法从 `org.springframework.core.MethodParameter` 中获取的

```java
private Type getReturnType(MethodParameter methodParameter) {
    Type returnType = Object.class;
    for (ReturnTypeParser returnTypeParser : returnTypeParsers) {
        if (returnType.getTypeName().equals(Object.class.getTypeName())) {
            returnType = returnTypeParser.getReturnType(methodParameter);
        }
        else
            break;
    }

    return returnType;
}
```

我们注意到这个方法最终是通过 `org.springdoc.core.ReturnTypeParser` 来完成它的功能，`returnTypeParsers` 是 `GenericResponseService` 的一个属性，在创建它的实例的构造方法中作为外部参数传入

```java
@Lazy(false)
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnProperty(name = SPRINGDOC_ENABLED, matchIfMissing = true)
@ConditionalOnBean(SpringDocConfiguration.class)
public class SpringDocWebMvcConfiguration {
    // ...

	@Bean
	@ConditionalOnMissingBean
	@Lazy(false)
	GenericResponseService responseBuilder(OperationService operationService, List<ReturnTypeParser> returnTypeParsers, SpringDocConfigProperties springDocConfigProperties, PropertyResolverUtils propertyResolverUtils) {
		return new GenericResponseService(operationService, returnTypeParsers, springDocConfigProperties, propertyResolverUtils);
	}

    // ...
}
```

而 `ReturnTypeParser` 的实例在 `SpringDocConfiguration` 中构建

```java
@Lazy(false)
@Configuration(proxyBeanMethods = false)
@ConditionalOnProperty(name = SPRINGDOC_ENABLED, matchIfMissing = true)
@ConditionalOnWebApplication
public class SpringDocConfiguration {
    // ...

	@Bean
	@Lazy(false)
	ReturnTypeParser genericReturnTypeParser() {
		return new ReturnTypeParser() {};
	}

    // ...
}
```

我们可以实现 `ReturnTypeParser` 接口并重写它的 `getReturnType` 方法在返回值类型不是 `Result` 时构建相应的 `Result.class` 实现我们的目标。

## 解决问题

根据问题分析的描述我们首先实现 `ReturnTypeParser` 接口并重写它的 `getReturnType` 方法

```java
package me.acomma.example.swagger;

import me.acomma.example.common.Result;
import org.apache.commons.lang3.reflect.TypeUtils;
import org.springdoc.core.ReturnTypeParser;
import org.springframework.core.MethodParameter;
import org.springframework.web.bind.annotation.RestController;

import java.lang.annotation.Annotation;
import java.lang.reflect.Type;
import java.util.Arrays;
import java.util.Objects;

public class ExampleReturnTypeParser implements ReturnTypeParser {
    @Override
    public Type getReturnType(MethodParameter methodParameter) {
        Type returnType = ReturnTypeParser.super.getReturnType(methodParameter);
        Annotation[] annotations = Objects.requireNonNull(methodParameter.getMethod()).getDeclaringClass().getAnnotations();

        if (Arrays.stream(annotations).noneMatch(annotation -> annotation instanceof RestController)) {
            return returnType;
        }

        // 如果返回值类型形如 Result<User>，那么 returnType 的类型为 ResolvableType.SyntheticParameterizedType，
        // 但是 ResolvableType.SyntheticParameterizedType 是私有静态类，外部无法访问，因此这里不能用 == 比较
        if (returnType.getTypeName().contains("me.acomma.example.common.Result")) {
            return returnType;
        }

        // returnType 在这里不会是类似 Result<Void> 的类型，也可以使用 returnType.getTypeName().equals("void") || returnType.getTypeName().equals("java.lang.Void")  进行判断，
        // 并且需要重新使用 Void.class 进行参数化，不然在后面会出现 NullPointerException
        if (returnType == void.class || returnType == Void.class) {
            return TypeUtils.parameterize(Result.class, Void.class);
        }

        return TypeUtils.parameterize(Result.class, returnType);
    }
}
```

我们需要注意最后一行代码，即 `TypeUtils.parameterize(Result.class, returnType)`，它实现了泛型类型参数化，可以参考参考资料[1]和[2]两篇文章了解学习。

然后把它注册到容器中

```java
package me.acomma.example.swagger;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ExampleSwaggerConfiguration {
    @Bean
    public ExampleReturnTypeParser exampleReturnTypeParser() {
        return new ExampleReturnTypeParser();
    }
}
```

看起来比上一篇文章中的实现简单直接多了，而且没有侵入 SpringDoc 的流程。

这个实现还有一点不完美，当返回值类型是 `void` 时，文档中不会有 Schema 部分。导致这个不完美的原因是 SpringDoc 会忽略返回值类型为 `void` 的方法的 Schema 构建。此时我们可以重写 `GenericResponseService` 类的 `buildContent` 方法来解决

```java
package me.acomma.example.swagger;

import com.fasterxml.jackson.annotation.JsonView;
import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.media.Content;
import io.swagger.v3.oas.models.media.MediaType;
import io.swagger.v3.oas.models.media.Schema;
import org.apache.commons.lang3.ArrayUtils;
import org.springdoc.core.GenericResponseService;
import org.springdoc.core.OperationService;
import org.springdoc.core.PropertyResolverUtils;
import org.springdoc.core.ReturnTypeParser;
import org.springdoc.core.SpringDocAnnotationsUtils;
import org.springdoc.core.SpringDocConfigProperties;

import java.lang.annotation.Annotation;
import java.lang.reflect.Type;
import java.util.Arrays;
import java.util.List;

import static org.springdoc.core.SpringDocAnnotationsUtils.extractSchema;

public class ExampleGenericResponseService extends GenericResponseService {
    /**
     * Instantiates a new Generic response builder.
     *
     * @param operationService          the operation builder
     * @param returnTypeParsers         the return type parsers
     * @param springDocConfigProperties the spring doc config properties
     * @param propertyResolverUtils     the property resolver utils
     */
    public ExampleGenericResponseService(OperationService operationService, List<ReturnTypeParser> returnTypeParsers, SpringDocConfigProperties springDocConfigProperties, PropertyResolverUtils propertyResolverUtils) {
        super(operationService, returnTypeParsers, springDocConfigProperties, propertyResolverUtils);
    }

    @Override
    public Content buildContent(Components components, Annotation[] annotations, String[] methodProduces, JsonView jsonView, Type returnType) {
        Content content = new Content();

        // 如果 returnType 是 void，父类在这里直接返回

        if (ArrayUtils.isNotEmpty(methodProduces)) {
            Schema<?> schemaN = calculateSchema(components, returnType, jsonView, annotations);
            if (schemaN != null) {
                io.swagger.v3.oas.models.media.MediaType mediaType = new io.swagger.v3.oas.models.media.MediaType();
                mediaType.setSchema(schemaN);
                // Fill the content
                setContent(methodProduces, content, mediaType);
            }
        }
        return content;
    }

    /**
     * @see GenericResponseService#calculateSchema(Components, Type, JsonView, Annotation[])
     */
    private Schema<?> calculateSchema(Components components, Type returnType, JsonView jsonView, Annotation[] annotations) {
        // 去掉了父类的 !isVoid(returnType) 判断
        if (!SpringDocAnnotationsUtils.isAnnotationToIgnore(returnType))
            return extractSchema(components, returnType, jsonView, annotations);
        return null;
    }

    /**
     * @see GenericResponseService#setContent(String[], Content, MediaType)
     */
    private void setContent(String[] methodProduces, Content content,
                            io.swagger.v3.oas.models.media.MediaType mediaType) {
        Arrays.stream(methodProduces).forEach(mediaTypeStr -> content.addMediaType(mediaTypeStr, mediaType));
    }
}
```

它只做了一件事情，就是把父类方法中的下面这个判断去掉了，其他都保持了不变

```java
if (isVoid(returnType))
    return null;
```

然后我们再它注册到容器中

```java
package me.acomma.example.swagger;

import org.springdoc.core.OperationService;
import org.springdoc.core.PropertyResolverUtils;
import org.springdoc.core.ReturnTypeParser;
import org.springdoc.core.SpringDocConfigProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class ExampleSwaggerConfiguration {
    @Bean
    public ExampleGenericResponseService exampleGenericResponseService(OperationService operationService, List<ReturnTypeParser> returnTypeParsers, SpringDocConfigProperties springDocConfigProperties, PropertyResolverUtils propertyResolverUtils) {
        return new ExampleGenericResponseService(operationService, returnTypeParsers, springDocConfigProperties, propertyResolverUtils);
    }

    @Bean
    public ExampleReturnTypeParser exampleReturnTypeParser() {
        return new ExampleReturnTypeParser();
    }
}
```

## 参考资料

1. [Spring-doc-openapi3实用配置](https://blog.csdn.net/catcher92/article/details/118926032)
2. [Java反射-基于ParameterizedType实现泛型类类型参数化](https://www.jianshu.com/p/b1ad2f1d3e3e)
3. [Java反射--基于ParameterizedType实现泛型类，参数化类型](https://www.cnblogs.com/sqy123/p/9707952.html)
