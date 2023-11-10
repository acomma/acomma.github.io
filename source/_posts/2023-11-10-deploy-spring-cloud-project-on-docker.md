---
title: 在 Docker 上部署 Spring Cloud 工程
date: 2023-11-10 23:25:04
tags:
---

我们有一个使用 Spring Cloud 技术栈开发的简单的商城项目 [acomma/shop](https://github.com/acomma/shop)，现在要将它构建成 Docker 镜像并以容器的方式运行起来。

<!-- more -->

## 创建并运行依赖容器

商城项目 [acomma/shop](https://github.com/acomma/shop) 使用到了 Nacos 和 MySQL，因此需要先在 Docker 中创建并运行对应的容器

```shell
# 创建并运行 Nacos 容器
docker run --name nacos -e MODE=standalone -p 8848:8848 -p 9848:9848 -p 9849:9849 -d nacos/nacos-server:v2.1.0

# 创建并运行 MySQL 容器
docker run -itd --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql:8.0.28
```

容器运行成功后在 Nacos 中创建配置（注意：MySQL 的 IP 地址替换为 MySQL 容器的名字 `mysql`），在 MySQL 中初始化表和数据。

## 创建自定义的容器网络

关于容器之间如何进行通信可以参考[每天5分钟玩转 Docker 容器技术](https://mp.weixin.qq.com/s/7o8QxGydMTUe4Q7Tz46Diw)第 5 章的内容，我们这里直接创建一个自定义网络 `shop-network`

```shell
docker network create --driver bridge --attachable shop-network
```

网络创建完成后将 Nacos 和 MySQL 容器连接到自定义网络

```shell
# 将名称为 nacos 的容器连接到自定义网络
docker network connect shop-network nacos

# 将名称为 mysql 的容器连接到自定义网络
docker network connect shop-network mysql
```

## 构建 Docker 镜像

在众多的构建插件中我们选择的是 [spotify/dockerfile-maven](https://github.com/spotify/dockerfile-maven)，选择的依据是在 [Spring Boot Docker](https://spring.io/guides/topicals/spring-boot-docker/) 中使用了它，其他的备选项还有 [fabric8io/docker-maven-plugin](https://github.com/fabric8io/docker-maven-plugin)。

注意：将 `spring.cloud.nacos.server-addr` 配置项的 IP 地址修改为 Nacos 容器的名称 `nacos`。

在项目的的父 `pom.xml` 文件中增加如下的配置

```xml
<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>com.spotify</groupId>
                <artifactId>dockerfile-maven-plugin</artifactId>
                <version>${dockerfile-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>default</id>
                        <goals>
                            <goal>build</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <repository>acomma/${project.artifactId}</repository>
                    <tag>${project.version}</tag>
                    <buildArgs>
                        <JAR_FILE>${project.build.finalName}.jar</JAR_FILE>
                    </buildArgs>
                </configuration>
            </plugin>
        </plugins>
    </pluginManagement>
</build>
```

在要构建 Docker 镜像的模块的 `pom.xml` 文件中增加如下内容

```xml
<build>
    <plugins>
        <plugin>
            <groupId>com.spotify</groupId>
            <artifactId>dockerfile-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

在要构建 Docker 镜像的模块的根目录中增加 `Dockerfile` 文件，内容如下

```dockerfile
FROM eclipse-temurin:17.0.6_10-jre-jammy
ARG JAR_FILE
COPY target/${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

然后命令行终端中运行如下命名构建 Docker 镜像

```shell
mvn -Dmaven.test.skip=true clean package
```

构建完成后执行如下的命令查询已经构建好的镜像

```shell
docker image ls
```

这条命令将显示如下的结果

```text
REPOSITORY                 TAG                       IMAGE ID       CREATED          SIZE
acomma/shop-user-web       0.0.1-SNAPSHOT            0249dfc01472   19 minutes ago   313MB
acomma/shop-product-web    0.0.1-SNAPSHOT            50ab8d01e89f   21 minutes ago   313MB
acomma/shop-order-web      0.0.1-SNAPSHOT            d2ec7c029c9a   22 minutes ago   325MB
acomma/shop-gateway        0.0.1-SNAPSHOT            ed0a947e878a   24 minutes ago   317MB
```

## 运行 Docker 镜像

创建 `docker-compose.yml` 文件，内容为

```yaml
version: "3.9"
services:
  user-web:
    container_name: shop-user-web
    image: acomma/shop-user-web:0.0.1-SNAPSHOT
    networks:
      - shop-network
    environment:
      - SPRING_PROFILES_ACTIVE=dev
  product-web:
    container_name: shop-product-web
    image: acomma/shop-product-web:0.0.1-SNAPSHOT
    networks:
      - shop-network
    environment:
      - SPRING_PROFILES_ACTIVE=dev
  order-web:
    container_name: shop-order-web
    image: acomma/shop-order-web:0.0.1-SNAPSHOT
    networks:
      - shop-network
    environment:
      - SPRING_PROFILES_ACTIVE=dev
  gateway:
    container_name: shop-gateway
    image: acomma/shop-gateway:0.0.1-SNAPSHOT
    ports:
      - "8080:8080"
    networks:
      - shop-network
    environment:
      - SPRING_PROFILES_ACTIVE=dev
networks:
  shop-network:
    external: true
```

需要注意 `networks` 的配置，这里使用网络 `shop-network` 是由前面的 `docker network create` 创建的，不是由 `docker compose` 创建的，因此 `external` 的值配置为 `true`。

然后执行 `docker compose up -d` 启动服务。

## 分布式链路追踪

我们选择使用 [Zipkin](https://zipkin.io/) 和 [Spring Cloud Sleuth](https://spring.io/projects/spring-cloud-sleuth) 实现分布式链路追踪。

首先我们需要在 Docker 上部署 Zipkin 并把它加入 shop-network 网络

```shell
docker run -itd -p 9411:9411 --name zipkin openzipkin/zipkin:2.24

docker network connect shop-network zipkin
```

然后在每个服务的 pom.xml 文件中引入相关依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>
```

最后在配置文件中 Zipkin 的配置

```yaml
spring:
  zipkin:
    base-url: http://zipkin:9411
    discovery-client-enabled: false
```

## 分布式日志收集

我们选择使用 [Elastic Stack](https://www.elastic.co/cn/elastic-stack/) 来搭建分布式日志系统。

第 1 步，修改 `docker-compose.yml` 文件，把日志目录映射到主机的目录上

```yaml
version: "3.9"
services:
    # 省略了...
    volumes:
      - type: bind
        source: /d/docker/shop/log/user
        target: /var/log/shop
  product-web:
    # 省略了...
    volumes:
      - type: bind
        source: /d/docker/shop/log/product
        target: /var/log/shop
  order-web:
    # 省略了...
    volumes:
      - type: bind
        source: /d/docker/shop/log/order
        target: /var/log/shop
  gateway:
    # 省略了...
    volumes:
      - type: bind
        source: /d/docker/shop/log/gateway
        target: /var/log/shop
# 省略了...
```

配置完成后重新打包发布应用。

第 2 步，下载 ElasticSearch、Kibana、Logstash、Filebeat

```shell
docker pull elasticsearch:7.10.1
docker pull kibana:7.10.1
docker pull logstash:7.10.1
docker pull elastic/filebeat:7.10.1
```

第 3 步，创建一个自定义网络 elastic-network

```shell
docker network create --driver bridge --attachable elastic-network
```

第 4 步，部署 Elasticsearch

参考官方文档 [Install Elasticsearch with Docker](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docker.html)。

```shell
docker run \
    -itd \
    -p 9200:9200 \
    -p 9300:9300 \
    --name elasticsearch \
    --network elastic-network \
    -e "cluster.name=elasticsearch" \
    -e "discovery.type=single-node" \
    -e "ES_JAVA_OPTS=-Xms512m -Xmx512m" \
    -e "LANG=C.UTF-8" \
    -e "LC_ALL=C.UTF-8" \
    -v /d/docker/elasticsearch/data:/usr/share/elasticsearch/data \
    -v /d/docker/elasticsearch/logs:/usr/share/elasticsearch/logs \
    -v /d/docker/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
    elasticsearch:7.10.1
```

根据 [Docker-compose 部署 ELK （单机版）详细教程](https://juejin.cn/post/7143974532766760990)的描述需要使用 `ES_JAVA_OPTS` 对 ES 容器的内存进行限制。

部署完成后在浏览器访问 [http://127.0.0.1:9200](http://127.0.0.1:9200) 验证是否成功，将显示类似下面的结果

```json
{
  "name" : "c5f21c919c84",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "iTevQv5ITjaxj0s1PyhHuQ",
  "version" : {
    "number" : "7.10.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "1c34507e66d7db1211f66f3513706fdf548736aa",
    "build_date" : "2020-12-05T01:00:33.671820Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

第 5 步，部署 Kibana

参考官方文档 [Install Kibana with Docker](https://www.elastic.co/guide/en/kibana/7.17/docker.html)。

```shell
docker run \
    -itd \
    -p 5601:5601 \
    --name kibana \
    --network elastic-network \
    -e "ELASTICSEARCH_HOSTS=http://elasticsearch:9200" \
    -e "I18N_LOCALE=zh-CN" \
    kibana:7.10.1
```

部署完成后在浏览器访问 [http://127.0.0.1:5601](http://127.0.0.1:5601) 验证是否成功。

第 6 步，部署 Logstash

参考官方文档 [Running Logstash on Docker](https://www.elastic.co/guide/en/logstash/7.17/docker.html)。

创建 `/d/docker/logstash/pipeline/logstash.conf` 文件，内容如下所示

```text
input {
    beats {
        port => 5044
        client_inactivity_timeout => 36000
    }
}

output {
    elasticsearch {
        hosts => ["elasticsearch:9200"]
        index => "shop"
    }
}
```

Logstash 默认包含了 Beats 插件，因此可以直接使用，参考[Filebeat和Logstash的简单配置和使用](https://www.jianshu.com/p/2f050b8ab859)。

执行如下的指令部署 Logstash 容器

```shell
docker run \
    -itd \
    -p 5044:5044 \
    -p 5045:5045 \
    --name logstash \
    --network elastic-network \
    -v /d/docker/logstash/pipeline/logstash.conf:/usr/share/logstash/pipeline/logstash.conf \
    logstash:7.10.1
```

这里要注意 `logstash.conf` 文件在容器中的路径，参考[基于 Docker 如何快速搭建 ELK？](https://www.zhihu.com/question/585232670/answer/2903005587)和[ELK 8.4.3 docker 保姆级安装部署详细步骤](https://blog.csdn.net/weixin_44259233/article/details/128456922)。

第 7 步，部署 Filebeat

参考官方文档 [Run Filebeat on Docker](https://www.elastic.co/guide/en/beats/filebeat/7.17/running-on-docker.html)。

创建 `/d/docker/filebeat/filebeat.yml` 文件，内容如下所示

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/shop/user/*.log
      - /var/log/shop/product/*.log
      - /var/log/shop/order/*.log
      - /var/log/shop/gateway/*.log
output.logstash:
  hosts: ["logstash:5044"]
```

Filebeat 将扫描 `paths` 配置的文件并把内容发给 Logstash。

执行如下的指令部署 Filebeat 容器

```shell
docker run \
    -itd \
    --name filebeat \
    --network elastic-network \
    -v /d/docker/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml \
    -v /d/docker/shop/log/user:/var/log/shop/user \
    -v /d/docker/shop/log/product:/var/log/shop/product \
    -v /d/docker/shop/log/order:/var/log/shop/order \
    -v /d/docker/shop/log/gateway:/var/log/shop/gateway \
    elastic/filebeat:7.10.1 -e -strict.perms=false
```

Filebeat 容器的启动命令需要加上 `-e -strict.perms=false`，不然会遇到 `filebeat.yml` 权限问题，参考 [filebeat构建的docker容器运行时提示需要权限怎么搞](https://blog.csdn.net/weixin_43702146/article/details/120889272)、[Volume mapped filebeat.yml permissions from Docker on a Windows host](https://stackoverflow.com/questions/44926280/volume-mapped-filebeat-yml-permissions-from-docker-on-a-windows-host) 和 [docker-部署filebeat](https://blog.csdn.net/ssehs/article/details/108917192)。

后面四个 `-v` 配置数据卷的意思是要把应用的日志目录映射到容器的目录，这样应用就 Filebeat 共享主机的 `D:\docker\shop\log` 目录。总结起来就是应用将日志写入主机的 `D:\docker\shop\log` 目录，Filebeat 读取这个目录下的文件，把内容发送给 Logstash，Logstash 进行一些处理后把结果发送给 ElasticSearch，Kibana 从 ES 中读取内容展示给用户。 

另外也可以使用[使用 Filebeat 收集 Docker 容器日志](https://zhaozhiming.github.io/2019/09/14/use-filebeat-to-collect-docker-container-log/)介绍的读取 `/var/lib/docker/containers` 目录。

到这里所需要的日志所需要的组件已经部署完成，访问一下应用使他们生成一些日志，等待一小会儿之后就可以在 Kibana 上看到 `shop` 索引，然后按照[logstash收集的日志输出到elasticsearch中](https://www.cnblogs.com/huan1993/p/15416088.html)的介绍创建索引模式就可以了。

## Jenkins 自动化部署

注意：在 Windows 下 `/var/run/docker.sork` 前面多一个 `/`，参考文章在[Running Docker in Docker on Windows (Linux containers)](https://tomgregory.com/running-docker-in-docker-on-windows/)
