---
title: Knife4j OpenAPI2 的一个隐藏限制：@ApiModel 的 value 千万别加点
date: 2026-05-01 23:35:09
updated: 2026-05-01 23:35:09
tags: [Knife4j, Swagger2]
---

## 问题描述

我们有一个项目使用了 `com.github.xiaoymin:knife4j-openapi2-spring-boot-starter:4.4.0` 依赖，API 接口的出/入参参数使用了内部类，举个例子

```java
@Data
@ApiModel(value = "Outer")
public class Outer {
    private String name;

    private Inner inner;

    @Data
    @ApiModel(value = "Outer.Inner")
    public static class Inner {
        private String name;
    }
}
```

注意第 9 行代码中 `value` 的值使用了点 `.` 来分割外部类与内部类的名称，当访问 API 文档时 `inner` 属性被当成了 `string` 类型而不是 `object` 类型

{% asset_img use-dot.png %}

<!-- more -->

## 原因分析

一开始怀疑是 Knife4j 文档显示问题，但是当调用 `/v2/api-docs` 接口查看生成文档的 JSON 数据时，发现第 7 行 `$ref` 属性缺少了 `#/definitions/` 前缀，导致 `$ref` 值变成了 `Outer.Inner` 而不是 `#/definitions/Outer.Inner`。因为 `$ref` 值是不完整的，Knife4j 无法通过它找到对应的模型定义，从而导致 `$ref` 值被解析成 `string` 类型。

```json
{
    "definitions": {
        "Outer": {
            "type": "object",
            "properties": {
                "inner": {
                    "$ref": "Outer.Inner",
                    "originalRef": "Outer.Inner"
                },
                "name": {
                    "type": "string"
                }
            },
            "title": "Outer"
        },
        "Outer.Inner": {
            "type": "object",
            "properties": {
                "name": {
                    "type": "string"
                }
            },
            "title": "Outer.Inner"
        }
    }
}
```

我们知道在 `io.swagger.models.Swagger` 中使用 `io.swagger.models.refs.GenericRef` 来描述 `$ref` 属性，在构造 `GenericRef` 对象时会调用 `GenericRef#computeRefFormat` 来确定引用的格式

```java
private static RefFormat computeRefFormat(String ref) {
    RefFormat result = RefFormat.INTERNAL;
    if (ref.startsWith("http:") || ref.startsWith("https:")) {
        result = RefFormat.URL;
    } else if (ref.startsWith("#/")) {
        result = RefFormat.INTERNAL;
    } else if (ref.startsWith(".") || ref.startsWith("/")) {
        result = RefFormat.RELATIVE;
    } else if (
            relativeRefWithAnyDot &&
            !ref.contains(":") &&   // No scheme
            !ref.startsWith("#") && // Path is not empty
            !ref.startsWith("/")&& // Path is not absolute
            ref.indexOf(".") > -1) {
        result = RefFormat.RELATIVE;
    }
    return result;
}
```

这个函数 `ref` 参数的值是在 `@ApiModel#value` 属性中指定的，在这里为 `Outer.Inner`。第 10~14 行的条件为 `true`，因此函数返回 `RefFormat.RELATIVE`。当调用回到 `GenericRef` 构造函数后

```java
public GenericRef(RefType type, String ref, RefFormat format) {
    this.originalRef = ref;
    if (format == null) {
        this.format = computeRefFormat(ref);
    } else {
        this.format = format;
    }

    this.type = type;

    if (this.format == RefFormat.INTERNAL && !ref.startsWith("#/")) {
        /* this is an internal path that did not start with a #/, we must be in some of ModelResolver code
        while currently relies on the ability to create RefModel/RefProperty objects via a constructor call like
        1) new RefModel("Animal")..and expects get$ref to return #/definitions/Animal
        2) new RefModel("http://blah.com/something/file.json")..and expects get$ref to turn the URL
            */
        this.ref = type.getInternalPrefix() + ref;
    } else {
        this.ref = ref;
    }

    this.simpleRef = computeSimpleRef(this.ref, this.format, type);
}
```

第 11 行的条件为 `false`，因此第 19 行 `this.ref` 的值被设置为 `Outer.Inner`。这就是在 `/v2/api-docs` 看到的结果。

然而使用和 Knife4j 相同版本的 `io.springfox:springfox-swagger2:2.10.5` 测试时，`$ref` 值被解析成 `#/definitions/Outer.Inner`，因此 `Outer.Inner` 被解析成 `object` 类型。分析两者的依赖关系发现 `com.github.xiaoymin:knife4j-openapi2-spring-boot-starter:4.4.0` 使用的 swagger-models 版本为 `io.swagger:swagger-models:1.6.6`，而前者使用的 swagger-models 版本为 `io.swagger:swagger-models:1.5.20`，在 `io.swagger:swagger-models:1.5.20` 中 `GenericRef#computeRefFormat` 函数的实现为

```java
private static RefFormat computeRefFormat(String ref) {
    RefFormat result = RefFormat.INTERNAL;
    if (ref.startsWith("http:") || ref.startsWith("https:")) {
        result = RefFormat.URL;
    } else if (ref.startsWith("#/")) {
        result = RefFormat.INTERNAL;
    } else if (ref.startsWith(".") || ref.startsWith("/")) {
        result = RefFormat.RELATIVE;
    }

    return result;
}
```

没有进行 `relativeRefWithAnyDot` 的判断，因此 `$ref` 值被解析成 `#/definitions/Outer.Inner`。

## 解决方案

原因讲清楚后解决这个问题就变得比较简单了。方法一是降级 swagger-models 的版本为 `io.swagger:swagger-models:1.5.20`。方法二是不使用内部类。方法三是使用内部类，但是 `value` 属性值不带点，比如 `@ApiModel(value = "Outer$Inner")` 或者 `@ApiModel(value = "OuterInner")`。

完~
