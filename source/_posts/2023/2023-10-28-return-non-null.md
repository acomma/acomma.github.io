---
title: 基于请求参数动态控制是否返回值为空的字段
date: 2023-10-28 17:39:02
updated: 2023-10-28 17:39:02
tags: [Spring Boot, Jackson]
---

## 问题描述

在 SpringBoot 应用中可以使用以下几种方式来控制是否返回值为空的字段

1. 在配置文件 `application.yml` 中配置全局自动忽略
    ```yml
    spring:
      jackson:
        default-property-inclusion: NON_NULL
    ```
2. 在类或字段上添加注解 `@JsonInclude(JsonInclude.Include.NON_NULL)`
3. 实现一个 `Jackson2ObjectMapperBuilderCustomizer` 定制器
    ```yml
    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        return jacksonObjectMapperBuilder -> {
            jacksonObjectMapperBuilder.serializationInclusion(JsonInclude.Include.NON_NULL);
        };
    }
    ```
4. 参考 `JacksonObjectMapperConfiguration` 自定义一个 `ObjectMapper`，覆盖默认的定义
    ```yml
    @Bean
    @Primary
    @ConditionalOnMissingBean
    public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
        ObjectMapper mapper = builder.createXmlMapper(false).build();
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        return mapper;
    }
    ```

这 4 种方式要么是全局的要么是局部的，它们都是一次性的，设置后就不能改变，每次请求都会返回相同的结果。现在我们要实现根据请求参数中是否包含某个值动态的设置是否返回值为空的字段，具体的说我们要根据请求头中是否包含 `X-Include-Non-Null` 来控制是否返回值为空的字段。

<!-- more -->

## 自定义 MappingJackson2HttpMessageConverter

首先我们需要实现一个拦截器，把请求中的 `X-Include-Non-Null` 放入响应头或者线程本地变量中，方便在后面获取这个值

```java
public class NullValueSerializationInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        response.setHeader("X-Include-Non-Null", request.getHeader("X-Include-Non-Null"));
        return true;
    }
}
```

然后把它加入拦截器集合中使它生效

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new NullValueSerializationInterceptor());
    }
}
```

接下来我们要自定义两个 `ObjectMapper`

```java
/**
 * 这是默认的 {@code ObjectMapper} 实现，
 * 拷贝自 {@link org.springframework.boot.autoconfigure.jackson.JacksonAutoConfiguration.JacksonObjectMapperConfiguration#jacksonObjectMapper(org.springframework.http.converter.json.Jackson2ObjectMapperBuilder)}，
 * 如果不拷贝，因为 Bean 定义上有 {@code @ConditionalOnMissingBean}，而且自定义了一个 {@link WebMvcConfig#customObjectMapper(Jackson2ObjectMapperBuilder)}，会导致
 * {@link MappingJackson2HttpMessageConverter#objectMapper} 属性值不对
 */
@Bean
@Primary
public ObjectMapper jacksonObjectMapper(Jackson2ObjectMapperBuilder builder) {
    ObjectMapper mapper = builder.createXmlMapper(false).build();
    return mapper;
}

/**
 * 自定义一个 {@code ObjectMapper}，它的创建过程与 {@link WebMvcConfig#jacksonObjectMapper(Jackson2ObjectMapperBuilder)} 一样
 */
@Bean
public ObjectMapper customObjectMapper(Jackson2ObjectMapperBuilder builder) {
    ObjectMapper mapper = builder.createXmlMapper(false).build();
    // WARNING：这是自定义的部分
    mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);

    // WARNING：如果有其他配置需要和 jacksonObjectMapper 中的配置过程保持一致

    return mapper;
}
```

现在我们需要实现自己的 `MappingJackson2HttpMessageConverter`

```java
import com.fasterxml.jackson.core.JsonEncoding;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.PrettyPrinter;
import com.fasterxml.jackson.core.util.DefaultIndenter;
import com.fasterxml.jackson.core.util.DefaultPrettyPrinter;
import com.fasterxml.jackson.databind.*;
import com.fasterxml.jackson.databind.exc.InvalidDefinitionException;
import com.fasterxml.jackson.databind.exc.MismatchedInputException;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import org.springframework.http.HttpOutputMessage;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConversionException;
import org.springframework.http.converter.HttpMessageNotWritableException;
import org.springframework.http.converter.json.MappingJackson2HttpMessageConverter;
import org.springframework.http.converter.json.MappingJacksonValue;
import org.springframework.util.TypeUtils;

import java.io.IOException;
import java.lang.reflect.Type;

public class CustomMappingJackson2HttpMessageConverter extends MappingJackson2HttpMessageConverter {
    private final ObjectMapper customObjectMapper;

    private final PrettyPrinter ssePrettyPrinter;

    public CustomMappingJackson2HttpMessageConverter(ObjectMapper jacksonObjectMapper, ObjectMapper customObjectMapper) {
        super(jacksonObjectMapper);
        this.customObjectMapper = customObjectMapper;

        // 下面这部分拷贝自父类
        DefaultPrettyPrinter prettyPrinter = new DefaultPrettyPrinter();
        prettyPrinter.indentObjectsWith(new DefaultIndenter("  ", "\ndata:"));
        this.ssePrettyPrinter = prettyPrinter;
    }

    // ---重写开始--- 和 ---重写结束--- 之间的部分与父类方法不一样，其他部分保持一致
    @Override
    protected void writeInternal(Object object, Type type, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {
        MediaType contentType = outputMessage.getHeaders().getContentType();
        JsonEncoding encoding = getJsonEncoding(contentType);

        // ---重写开始---
        JsonGenerator generator;
        if (includeNonNull(outputMessage)) {
            generator = getJacksonObjectMapper().getFactory().createGenerator(outputMessage.getBody(), encoding);
        } else {
            generator = getCustomObjectMapper().getFactory().createGenerator(outputMessage.getBody(), encoding);
        }
        // ---重写结束---

        try {
            writePrefix(generator, object);

            Object value = object;
            Class<?> serializationView = null;
            FilterProvider filters = null;
            JavaType javaType = null;

            if (object instanceof MappingJacksonValue) {
                MappingJacksonValue container = (MappingJacksonValue) object;
                value = container.getValue();
                serializationView = container.getSerializationView();
                filters = container.getFilters();
            }
            if (type != null && TypeUtils.isAssignable(type, value.getClass())) {
                javaType = getJavaType(type, null);
            }

            // ---重写开始---
            ObjectWriter objectWriter;
            if (includeNonNull(outputMessage)) {
                objectWriter = (serializationView != null ?
                        getJacksonObjectMapper().writerWithView(serializationView) : getJacksonObjectMapper().writer());
            } else {
                objectWriter = (serializationView != null ?
                        getCustomObjectMapper().writerWithView(serializationView) : getCustomObjectMapper().writer());
            }
            // ---重写结束---

            if (filters != null) {
                objectWriter = objectWriter.with(filters);
            }
            if (javaType != null && javaType.isContainerType()) {
                objectWriter = objectWriter.forType(javaType);
            }
            SerializationConfig config = objectWriter.getConfig();
            if (contentType != null && contentType.isCompatibleWith(MediaType.TEXT_EVENT_STREAM) &&
                    config.isEnabled(SerializationFeature.INDENT_OUTPUT)) {
                objectWriter = objectWriter.with(this.ssePrettyPrinter);
            }
            objectWriter.writeValue(generator, value);

            writeSuffix(generator, object);
            generator.flush();
        } catch (MismatchedInputException ex) {  // specific kind of JsonMappingException
            throw new HttpMessageNotWritableException("Invalid JSON input: " + ex.getOriginalMessage(), ex);
        } catch (InvalidDefinitionException ex) {  // another kind of JsonMappingException
            throw new HttpMessageConversionException("Type definition error: " + ex.getType(), ex);
        } catch (JsonMappingException ex) {  // typically ValueInstantiationException
            throw new HttpMessageConversionException("JSON mapping problem: " + ex.getPathReference(), ex);
        } catch (JsonProcessingException ex) {
            throw new HttpMessageNotWritableException("Could not write JSON: " + ex.getOriginalMessage(), ex);
        }
    }

    public ObjectMapper getCustomObjectMapper() {
        return customObjectMapper;
    }

    public ObjectMapper getJacksonObjectMapper() {
        return super.getObjectMapper();
    }

    public boolean includeNonNull(HttpOutputMessage outputMessage) {
        String includeNonNull = outputMessage.getHeaders().getFirst("X-Include-Non-Null");
        return includeNonNull == null;
    }
}
```

最后我们把它加入 Spring 容器中

```java
/**
 * 定义参考了 {@link org.springframework.boot.autoconfigure.http.JacksonHttpMessageConvertersConfiguration.MappingJackson2HttpMessageConverterConfiguration#mappingJackson2HttpMessageConverter(ObjectMapper)}
 */
@Bean
public CustomMappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter(@Qualifier("jacksonObjectMapper") ObjectMapper jacksonObjectMapper, @Qualifier("customObjectMapper") ObjectMapper customObjectMapper) {
    return new CustomMappingJackson2HttpMessageConverter(jacksonObjectMapper, customObjectMapper);
}
```

## 自定义 ResponseBodyAdvice

到这里基本就结束了，但是我们还有另外一种办法可以解决这个问题，只需要加入一个 `ResponseBodyAdvice` 的实现类即可

```java
import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.MethodParameter;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.server.ServerHttpRequest;
import org.springframework.http.server.ServerHttpResponse;
import org.springframework.util.ClassUtils;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseBodyAdvice;

import java.util.Iterator;
import java.util.Map;

@RestControllerAdvice
public class CustomResponseBodyAdvice implements ResponseBodyAdvice<Object> {
    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType) {
        return true;
    }

    @Override
    public Object beforeBodyWrite(Object body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response) {
        if (body == null || ClassUtils.isPrimitiveOrWrapper(body.getClass()) || body instanceof String) {
            return body;
        }

        if (includeNonNull(request)) {
            return body;
        }

        try {
            String content = objectMapper.writeValueAsString(body);
            JsonNode jsonNode = objectMapper.readTree(content);
            trimNull(jsonNode, 0);
            return jsonNode;
        } catch (JsonProcessingException e) {
            return body;
        }
    }

    /**
     * 删除值为NULL的字段
     */
    private void trimNull(JsonNode jsonNode, int depth) {
        if (jsonNode == null || depth >= 2) {
            return;
        }

        if (jsonNode.isObject()) {
            ObjectNode objectNode = (ObjectNode) jsonNode;
            Iterator<Map.Entry<String, JsonNode>> iterator = objectNode.fields();
            while (iterator.hasNext()) {
                Map.Entry<String, JsonNode> entry = iterator.next();
                if (entry.getValue().isNull()) {
                    iterator.remove();
                } else {
                    if (entry.getValue().isArray() || entry.getValue().isObject()) {
                        trimNull(entry.getValue(), depth + 1);
                    }
                }
            }
        } else if (jsonNode.isArray()) {
            ArrayNode arrayNode = (ArrayNode) jsonNode;
            for (JsonNode node : arrayNode) {
                trimNull(node, depth + 1);
            }
        }
    }

    /**
     * 是否忽略值为NULL的字段
     */
    public boolean includeNonNull(ServerHttpRequest request) {
        String includeNonNull = request.getHeaders().getFirst("X-Include-Non-Null");
        return includeNonNull != null;
    }
}
```

这种方式改动较小，效率可能要慢一点，无伤大雅。
