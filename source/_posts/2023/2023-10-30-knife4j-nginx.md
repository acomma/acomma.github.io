---
title: 解决通过 Nginx 无法打开 Knife4j 页面问题
date: 2023-10-30 21:29:43
updated: 2023-10-30 21:29:43
tags: [Knife4j, Nginx]
---

## 问题描述

我们有一个项目使用了 Knife4j，依赖的版本为

```xml
<dependency>
    <groupId>com.github.xiaoymin</groupId>
    <artifactId>knife4j-openapi3-jakarta-spring-boot-starter</artifactId>
    <version>4.3.0</version>
</dependency>
```

这个版本的 Knife4j 使用了 SpringDoc，其获取 Swagger 配置的地址为 `/v3/api-docs/swagger-config`，获取文档的地址为 `/v3/api-docs`，如果项目有多个分组，获取单个分组，比如分组“1-动物”，的文档的地址为 `/v3/api-docs/1-动物`。

在项目部署时在项目的前面放置了一台 Nginx 反向代理服务器，配置了当访问路径为 `/example` 时会将请求转发到我们自己的项目

```text
location /example/ {
    proxy_pass http://localhost:8080/;
}
```

当访问 `http://localhost/example/doc.html` 时，Knife4j 能正常的访问 Swagger 的配置，即 `http://localhost/example/v3/api-docs/swagger-config`，但是无法访问 `http://localhost/v3/api-docs/1-动物`，因此无法打开 Knife4j 文档页面。

<!-- more -->

## 分析问题

我们打开 Knife4j 的前端工程 `knife4j-vue`，在 `BasicLayout.vue` 在挂载完成后会调用 `created()` 方法

```js
created() {
    this.initSpringDocOpenApi();
    // 省略了...
}
```

因为我们使用的是 SpringDoc，它将调用 `initSpringDocOpenApi()` 方法

```json
initSpringDocOpenApi() {
    // 省略了...
    this.$localStore.getItem(constant.globalSettingsKey).then(settingCache => {
        // 省略了...
        this.$localStore.getItem(constant.globalGitApiVersionCaches).then(gitVal => {
            // 省略了...
            if (i18nParams.include) {
                // 省略了...
                this.initSwagger({
                    springdoc: true,
                    // 省略了...
                })
            } else {
                this.$localStore.getItem(constant.globalI18nCache).then(i18n => {
                    // 省略了...
                    this.initSwagger({
                        springdoc: true,
                        // 省略了...
                    })
                })
            }
        })
    })
}
```

在这个方法中会调用 `initSwagger()` 方法，这个方法的参数重有一个比较重要的是 `springdoc: true`

```js
initSwagger(options) {
    // 省略了...
    var swagger = new SwaggerBootstrapUi(options);
    try {
        swagger.main();
        // 省略了...
    } catch (e) {
        console.error(e);
    }
    // 省略了...
}
```

在 `initSwagger()` 方法中会创建 `SwaggerBootstrapUi` 对象，然后调用 `main()` 方法。

`SwaggerBootstrapUi` 对象定义在 `Knife4jAsync.js` 文件中，它的定义如下

```js
function SwaggerBootstrapUi(options) {
    // 省略了...
    this.springdoc = options.springdoc || false;
    if (this.springdoc) {
        const path = window.location.pathname;
        const index = path.lastIndexOf('/');
        const basePath = path.length == index + 1 ? path : path.substring(0, index);
        if (basePath != '' && basePath != '/') {
            this.url = options.url || basePath + '/v3/api-docs/swagger-config';
        } else {
            this.url = options.url || 'v3/api-docs/swagger-config';
        }
    } else {
        // 省略了...
    }
    // 省略了...
}
```

我们看到在 `this.springdoc` 的值为 `true` 时，会从浏览器地址 `http://localhost/example/doc.html` 中截取出 `http://localhost/example/` 部分作为 `basePath`，然后拼接上 `v3/api-docs/swagger-config` 部分得到 `this.url` 的值，即 `http://localhost/example/v3/api-docs/swagger-config`。这也就解释了为什么获取 Swagger 的配置没有问题。

从前面的分析我们知道在创建了 `SwaggerBootstrapUi` 对象后会调用它的 `main()` 方法，我们跟踪它的方法调用链直到 `analysisGroup()` 方法，在这个方法中调用前面组装的 `this.url`，即 `http://localhost/example/v3/api-docs/swagger-config` 获取分组信息

```js
SwaggerBootstrapUi.prototype.analysisGroup = function () {
  // 省略了...
  try {
    // 省略了...
    that.ajax({
      url: that.url,
      // 省略了...
    }, data => {
      if (that.springdoc) {
        that.analysisSpringDocOpenApiGroupSuccess(data);
      } else {
        // 省略了...
      }
      that.createGroupElement();
    }, err => {
      // 省略了...
    })
  } catch (err) {
    // 省略了...
  }
}
```

使用 `this.url` 作为参数调用 `ajax()` 方法获取的数据样本如下

```json
{
    "configUrl": "/v3/api-docs/swagger-config",
    "oauth2RedirectUrl": "http://xxx.yyy:8080/swagger-ui/oauth2-redirect.html",
    "operationsSorter": "order",
    "tagsSorter": "order",
    "urls": [
        {
            "url": "/v3/api-docs/1-动物",
            "name": "1-动物"
        },
        {
            "url": "/v3/api-docs/2-植物",
            "name": "2-植物"
        }
    ],
    "validatorUrl": ""
}
```

在 `analysisSpringDocOpenApiGroupSuccess()` 中解析分组信息

```js
SwaggerBootstrapUi.prototype.analysisSpringDocOpenApiGroupSuccess = function (data) {
    // 省略了...
    var t = typeof data;
    var groupData = null;
    if (t == 'string') {
        groupData = KUtils.json5parse(data);
    } else {
        groupData = data;
    }
    // 省略了...
    var groupUrls = KUtils.getValue(groupData, 'urls', [], true);
    var newGroupData = [];
    if (KUtils.arrNotEmpty(groupUrls)) {
        groupUrls.forEach(gu => {
            var newGroup = {
                // 省略了...
                url: KUtils.getValue(gu, 'url', '', true),
                location: KUtils.getValue(gu, 'url', '', true),
                // 省略了...
            };
            newGroupData.push(newGroup);
        })
    } else {
        // 省略了...
    }
    newGroupData.forEach(function (group) {
        var g = new SwaggerBootstrapUiInstance(
            KUtils.toString(group.name, '').replace(/\//g, '-'),
            group.location,
            group.swaggerVersion
        );
        g.url = group.url;
        // 省略了...
        g.servicePath = KUtils.getValue(group, 'servicePath', null, true);
        g.contextPath = KUtils.getValue(group, 'contextPath', null, true);
        var newUrl = '';
        if (group.url != null && group.url != undefined && group.url != '') {
            newUrl = group.url;
        } else {
            newUrl = group.location;
        }
        g.extUrl = newUrl;
        if (that.validateExtUrl == '') {
            that.validateExtUrl = g.extUrl;
        }
        if (group.basePath != null && group.basePath != undefined && group.basePath != '') {
            g.baseUrl = group.basePath;
        }
        // 省略了...
        that.instances.push(g);
    })
    // 省略了...
}
```

这个方法比较长，只保留了我们关心的内容，做了这么几件事情，从返回的信息中 `urls` 中解析出 `url`，构建 `SwaggerBootstrapUiInstance` 对象信息。

接下来 `analysisGroup()` 方法会调用 `createGroupElement()` 方法

```js
SwaggerBootstrapUi.prototype.createGroupElement = function () {
    // 省略了...
    var urlParams = this.routeParams;
    if (KUtils.checkUndefined(urlParams)) {
        if (urlParams.hasOwnProperty('groupName')) {
            var gpName = urlParams.groupName;
            if (KUtils.checkUndefined(gpName) && gpName != '') {
                let selectInstance = that.selectInstanceByGroupName(gpName);
                // 省略了...
                that.analysisApi(selectInstance);
            } else {
                that.analysisApi(that.instances[0]);
            }
        } else {
            that.analysisApi(that.instances[0]);
        }
    } else {
        that.analysisApi(that.instances[0]);
    }
}
```

这个方法会使用 `analysisGroup()` 方法构建的第一个 `SwaggerBootstrapUiInstance` 对象，用它调用 `analysisApi()` 方法

```js
SwaggerBootstrapUi.prototype.analysisApi = function (instance) {
    // 省略了...
    try {
        // 赋值
        that.currentInstance = instance;
        if (!that.currentInstance.load) {
            var api = instance.url;
            if (api == undefined || api == null || api == '') {
                api = instance.location;
            }
            if (that.settings.enableSwaggerBootstrapUi) {
                api = instance.extUrl;
            }
            // 省略了...
            var requestConfig = {
                url: api,
                // 省略了...
            };
            // 省略了...
            that.ajax(requestConfig, data => {
                that.analysisApiSuccess(data);
            }, err => {
                // 省略了...
            })
        } else {
            // 省略了...
        }
    } catch (err) {
        // 省略了...
    }
}
```

这个方法使用 `SwaggerBootstrapUiInstance` 对象中的 `url`，即 `/v3/api-docs/1-动物`，自动拼接上域名/端口部分，即 `http://loacalhost`，组成完整的地址 `http://loacalhost/v3/api-docs/1-动物` 获取文档信息。

到这里我们就可以看到从 `analysisSpringDocOpenApiGroupSuccess` 到 `createGroupElement` 到 `analysisApi` 没有一个地方像 `SwaggerBootstrapUi()` 方法中一样去获取或构建 `basePath` 部分，即 `http://loacalhost/example/`，然后拼接 `v3/api-docs/1-动物`。

## 解决问题

那么我们怎么解决这个问题呢？一种方法是修改 `knife4j-vue` 的源码然后重新打包。这里介绍另外一种方法。

编译打包后的 Knife4j 的 `doc.html` 文件位于 `knife4j-openapi3-ui` 包下面，它引入了两个文件 `webjars/js/app.51033393.js` 和 `webjars/js/chunk-vendors.d51cf6f8.js`，它们都位于 `knife4j-openapi3-ui` 包下面。Vue 编译打包后的方法名不会变更，从上面的分析我们知道可以在 `analysisSpringDocOpenApiGroupSuccess()` 方法解析分组信息时把 `basePath` 拼接在 `url` 的前面。在 `webjars/js/app.51033393.js` 文件中可以找到这个方法，我们把它手动格式化一下

```js
ze.prototype.analysisSpringDocOpenApiGroupSuccess=function(e){
    var t,n=this,a=Object(ie.a)(e);
    
    t="string"==a?re.a.json5parse(e):e,n.log("响应分组json数据"),n.log(t);
    
    var r=[],i=[],s=re.a.getValue(t,"urls",[],!0),o=[];
    
    re.a.arrNotEmpty(s)
    ?s.forEach((function(e){
        var n={
            name:re.a.getValue(e,"name","knife4j",!0),
            url:re.a.getValue(e,"url","",!0),
            location:re.a.getValue(e,"url","",!0),
            swaggerVersion:"3.0.3",
            tagSort:re.a.getValue(t,"tagsSorter","order",!0),
            operationSort:re.a.getValue(t,"operationsSorter","order",!0),
            servicePath:re.a.getValue(e,"servicePath",null,!0),
            contextPath:re.a.getValue(e,"contextPath",null,!0)
        };
        o.push(n)
    }))
    :o.push({
        name:re.a.getValue(t,"url","default",!0),
        url:re.a.getValue(t,"url","",!0),
        location:re.a.getValue(t,"url","",!0),
        swaggerVersion:"3.0.3",
        tagSort:re.a.getValue(t,"tagsSorter","order",!0),
        operationSort:re.a.getValue(t,"operationsSorter","order",!0),
        servicePath:re.a.getValue(t,"servicePath",null,!0),
        contextPath:re.a.getValue(t,"contextPath",null,!0)
    }),
    
    o.forEach((function(e){var t=new ft(re.a.toString(e.name,"").replace(/\//g,"-"),e.location,e.swaggerVersion);t.url=e.url,t.desktop=n.desktop,t.desktopCode=n.desktopCode,t.tagSort=e.tagSort,t.operationSort=e.operationSort,t.servicePath=re.a.getValue(e,"servicePath",null,!0),t.contextPath=re.a.getValue(e,"contextPath",null,!0);var a;if(a=null!=e.url&&null!=e.url&&""!=e.url?e.url:e.location,t.extUrl=a,""==n.validateExtUrl&&(n.validateExtUrl=t.extUrl),null!=e.basePath&&null!=e.basePath&&""!=e.basePath&&(t.baseUrl=e.basePath),n.cacheApis.length>0){var s=null;n.cacheApis.forEach((function(e){e.id==t.groupId&&(s=e)})),null!=s?(t.firstLoad=!1,s.hasOwnProperty("updateApis")||(s.updateApis={}),t.cacheInstance=s,n.log(t)):t.cacheInstance=new it({id:t.groupId,name:t.name})}else t.cacheInstance=new it({id:t.groupId,name:t.name});r.push({label:t.name,value:t.id}),i.push(t.id),n.instances.push(t)})),
    
    re.a.arrNotEmpty(n.instances)&&n.instances.forEach((function(e){e.allGroupIds=i})),
    
    this.serviceOptions=r,
    this.store.dispatch("globals/setServiceOptions",r),
    r.length>0&&(this.defaultServiceOption=r[0].value,this.store.dispatch("globals/setDefaultService",r[0].value))
}
```

参考 `SwaggerBootstrapUi` 构造方法中的做法，从浏览器地址中截取拼接 `basePath`

```js
const path = window.location.pathname;
const index = path.lastIndexOf('/');
const basePath = path.length == index + 1 ? path : path.substring(0, index);
```

把它们加入 `analysisSpringDocOpenApiGroupSuccess()` 方法中，然后修改 `url` 和 `location` 属性的赋值，加上 `basePath` 前缀

```js
ze.prototype.analysisSpringDocOpenApiGroupSuccess=function(e){
    var t,n=this,a=Object(ie.a)(e);
    
    t="string"==a?re.a.json5parse(e):e,n.log("响应分组json数据"),n.log(t);
    
    var r=[],i=[],s=re.a.getValue(t,"urls",[],!0),o=[];
    
    var path = window.location.pathname, index = path.lastIndexOf('/'), basePath = path.length == index + 1 ? path : path.substring(0, index);
    
    re.a.arrNotEmpty(s)
    ?s.forEach((function(e){
        var n={
            name:re.a.getValue(e,"name","knife4j",!0),
            url:basePath + re.a.getValue(e,"url","",!0),
            location:basePath + re.a.getValue(e,"url","",!0),
            swaggerVersion:"3.0.3",
            tagSort:re.a.getValue(t,"tagsSorter","order",!0),
            operationSort:re.a.getValue(t,"operationsSorter","order",!0),
            servicePath:re.a.getValue(e,"servicePath",null,!0),
            contextPath:re.a.getValue(e,"contextPath",null,!0)
        };
        o.push(n)
    }))
    // 省略了...
}
```

然后把修改后的文件按照 webjar 的方式放到自己项目的目录中，即自己项目的 `resources/webjars/js` 目录下，重启项目后就可以正常打开 Knife4j 的文档页面的，问题得到解决。
