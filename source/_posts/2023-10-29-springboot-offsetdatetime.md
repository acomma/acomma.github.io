---
title: 如何在 Spring Boot 中传递 OffsetDateTime 类型的参数？
date: 2023-10-29 17:25:14
tags:
---

## 问题描述

首先创建一个 Spring Boot 项目，然后创建一个实体类 `User`，它有两个字段姓名 `name` 和生日 `birthday`，其中 `birthday` 的类型是 `OffsetDataTime`

```java
@Data
public class User {
    private String name;
    private OffsetDateTime birthday;
}
```

最后创建对应的控制器 `UserController`，GET 请求的第二种形式将参数封装成了对象

```java
@RestController
@RequestMapping("/user")
public class UserController {
    @GetMapping("/query1")
    public OffsetDateTime query1(OffsetDateTime birthday) {
        return birthday;
    }

    @GetMapping("/query2")
    public User query2(User user) {
        return user;
    }

    @PostMapping("/create")
    public User create(@RequestBody User user) {
        return user;
    }
}
```

我们要实现接收前端传递过来的形如 `yyyy-MM-dd HH:mm:ss` 的字符串并自动将它转换为能够使用 `OffsetDateTime` 类型。

<!-- more -->

Spring Boot 默认支持 ISO-8601 格式，比如 `2007-12-03T10:15:30+01:00`，所以在请求时可以使用类似如下的格式

```shell
curl --location 'http://localhost:8080/user/query1?birthday=2023-09-02T12%3A30%3A03Z'

curl --location 'http://localhost:8080/user/query2?name=bob&birthday=2023-09-02T12%3A30%3A03Z'

curl --location 'http://localhost:8080/user/create' \
--header 'Content-Type: application/json' \
--data '{
    "birthday": "2023-09-02T12:30:03Z",
    "name": "bob"
}'
```

如果将 `birthday` 的值换成 `2023-09-02 12:30:05` 将会出现如下的异常，三行日志依次对应前面的三次请求

```text
2023-09-02T15:30:18.396+08:00  WARN 20588 --- [nio-8080-exec-8] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.web.method.annotation.MethodArgumentTypeMismatchException: Failed to convert value of type 'java.lang.String' to required type 'java.time.OffsetDateTime'; Failed to convert from type [java.lang.String] to type [java.time.OffsetDateTime] for value [2023-09-02 12:30:05]]
2023-09-02T15:30:28.181+08:00  WARN 20588 --- [nio-8080-exec-7] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.web.bind.MethodArgumentNotValidException: Validation failed for argument [0] in public com.example.springboot.entity.User com.example.springboot.controller.UserController.query2(com.example.springboot.entity.User): [Field error in object 'user' on field 'birthday': rejected value [2023-09-02 12:30:05]; codes [typeMismatch.user.birthday,typeMismatch.birthday,typeMismatch.java.time.OffsetDateTime,typeMismatch]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [user.birthday,birthday]; arguments []; default message [birthday]]; default message [Failed to convert property value of type 'java.lang.String' to required type 'java.time.OffsetDateTime' for property 'birthday'; Failed to convert from type [java.lang.String] to type [java.time.OffsetDateTime] for value [2023-09-02 12:30:05]]] ]
2023-09-02T15:30:34.475+08:00  WARN 20588 --- [io-8080-exec-10] .w.s.m.s.DefaultHandlerExceptionResolver : Resolved [org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Cannot deserialize value of type `java.time.OffsetDateTime` from String "2023-09-02 12:30:05": Failed to deserialize java.time.OffsetDateTime: (java.time.format.DateTimeParseException) Text '2023-09-02 12:30:05' could not be parsed at index 10]
```

现在我们就来实现对 `yyyy-MM-dd HH:mm:ss` 格式的支持。

## GET 请求

### 使用 Converter 的方式

首先需要实现一个转换器

```java
public class OffsetDateTimeConverter implements Converter<String, OffsetDateTime> {
    @Override
    public OffsetDateTime convert(String source) {
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        LocalDateTime localDateTime = LocalDateTime.parse(source, formatter);
        OffsetDateTime offsetDateTime = OffsetDateTime.of(localDateTime, ZoneOffset.UTC);
        return offsetDateTime;
    }
}
```

然后把自定义的转换器加入格式化注册中心中

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new OffsetDateTimeConverter());
    }
}
```

现在使用 `2023-09-02 12:30:05` 进行请求就不会出错了

```shell
curl --location 'http://localhost:8080/user/query1?birthday=2023-09-02%2012%3A30%3A05'

curl --location 'http://localhost:8080/user/query2?birthday=2023-09-02%2012%3A30%3A05&name=bob'
```

### 使用 InitBinder 的方式

这种方式只需要实现一个 `InitBinder` 方法即可

```java
@ControllerAdvice
public class WebDataBinderAdvice {
    @InitBinder
    public void initBinder(WebDataBinder binder) {
        binder.registerCustomEditor(OffsetDateTime.class, new PropertyEditorSupport() {
            @Override
            public void setAsText(String text) throws IllegalArgumentException {
                DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
                LocalDateTime localDateTime = LocalDateTime.parse(text, formatter);
                OffsetDateTime offsetDateTime = OffsetDateTime.of(localDateTime, ZoneOffset.UTC);
                super.setValue(offsetDateTime);
            }
        });
    }
}
```

`Converter` 和 `InitBinder` 都会由 `ModelAttributeMethodProcessor` 方法进行调用。

### 使用 AbstractNamedValueMethodArgumentResolver 的方式

我们先来实现一个 `HandlerMethodArgumentResolver` 类型的解析器看看效果怎么样？

```java
public class OffsetDateTimeArgumentResolver implements HandlerMethodArgumentResolver {
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType().equals(OffsetDateTime.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        String text = webRequest.getParameter("birthday");
        if (text != null) {
            try {
                DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
                LocalDateTime localDateTime = LocalDateTime.parse(text, formatter);
                OffsetDateTime offsetDateTime = OffsetDateTime.of(localDateTime, ZoneOffset.UTC);
                return offsetDateTime;
            } catch (Exception e) {
                // 处理解析错误，可以根据需要进行日志记录或返回默认值
            }
        }
        // 返回默认值或抛出异常，视情况而定
        return null; // 或者抛出异常
    }
}
```

然后把它加到方法参数解析器列表中

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new OffsetDateTimeArgumentResolver());
    }
}
```

这种方式对 `public OffsetDateTime query1(OffsetDateTime birthday) {}` 这种是有效的，但是对 `public User query2(User user) {}` 这种是无效的。

可以参考 `RequestParamMethodArgumentResolver` 的方式对 `OffsetDateTimeArgumentResolver` 进行增强，让它不仅仅处理名称为 `birthday` 的参数，还可以处理其他名称的类型为 `OffsetDateTime` 的参数，比如

```java
public class OffsetDateTimeArgumentResolver extends AbstractNamedValueMethodArgumentResolver {
    @Override
    protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
        RequestParam ann = parameter.getParameterAnnotation(RequestParam.class);
        if (ann == null) {
            return new NamedValueInfo("", false, ValueConstants.DEFAULT_NONE);
        }
        return new NamedValueInfo(ann.name(), ann.required(), ann.defaultValue());
    }

    @Override
    protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
        String[] paramValues = request.getParameterValues(name);
        if (paramValues == null) {
            return null;
        }
        String text = paramValues[0];
        if (text == null || text.isBlank()) {
            return null;
        }
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        LocalDateTime localDateTime = LocalDateTime.parse(text, formatter);
        OffsetDateTime offsetDateTime = OffsetDateTime.of(localDateTime, ZoneOffset.UTC);
        return offsetDateTime;
    }

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.getParameterType().equals(OffsetDateTime.class);
    }
}
```

现在就可以处理如下方法定义的请求了

```java
@GetMapping("/query3")
public OffsetDateTime query3(OffsetDateTime birth, String name, OffsetDateTime memo) {
    return birth;
}
```

因为 `resolveName` 方法有 `MethodParameter` 类型，因此也可以处理各种注解，比如根据 `DateTimeFormat` 注解处理自定义的时间格式。

## POST 请求

首先实现一个反序列化器

```java
public class OffsetDateTimeDeserializer extends JsonDeserializer<OffsetDateTime> {
    @Override
    public OffsetDateTime deserialize(JsonParser jsonParser, DeserializationContext deserializationContext) throws IOException {
        String text = jsonParser.getText();
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        LocalDateTime localDateTime = LocalDateTime.parse(text, formatter);
        OffsetDateTime offsetDateTime = OffsetDateTime.of(localDateTime, ZoneOffset.UTC);
        return offsetDateTime;
    }
}
```

然后向容器中加入一个 `Jackson2ObjectMapperBuilderCustomizer` 类型的 Bean

```java
@Configuration
public class JacksonConfig {
    @Bean
    public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
        return builder -> {
            JavaTimeModule module = new JavaTimeModule();
            module.addDeserializer(OffsetDateTime.class, new OffsetDateTimeDeserializer());
            builder.modules(module);
        };
    }
}
```

现在使用 `2023-09-02 12:30:05` 进行请求就不会出错了

```shell
curl --location 'http://localhost:8080/user/create' \
--header 'Content-Type: application/json' \
--data '{
    "birthday": "2023-09-02 12:30:05",
    "name": "bob"
}'
```

## 参考资料

1. [ZonedDateTime和OffsetDateTime之间的差异](https://tu-yucheng.github.io/java-new/2023/06/09/java-zoneddatetime-offsetdatetime.html)
2. [时区和偏移类 / Zone and Offset](https://zq99299.github.io/java-tutorial/datetime/iso/timezones.html)
3. [Spring Boot 日期时间处理总结，写的太好了。。](https://developer.aliyun.com/article/1052944)
4. [LocalDateTime、OffsetDateTime、ZonedDateTime互转，这一篇绝对喂饱你](https://fangshixiang.blog.csdn.net/article/details/112732546)
5. [JAVA中计算两个日期时间的差值竟然也有这么多门道](https://blog.csdn.net/weixin_45536242/article/details/125691352)
6. [彻底解决Spring mvc中时间的转换和序列化等问题](https://blog.csdn.net/qq_35067322/article/details/100990958)
7. [都什么年代了你还在用 Date](https://zhuanlan.zhihu.com/p/435892905)
8. [拨开时间的迷雾](https://zhuanlan.zhihu.com/p/269427524)
