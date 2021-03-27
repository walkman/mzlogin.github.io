---
layout: post
title: SpringCloud bus 消息总线 - 基于RocketMQ
categories: [SpringCloud,RocketMQ]
excerpt: spring cloud bus整合java的事件处理机制和消息中间件的发送和接收，主要是由发送端、接收端和事件组成。
keywords: ElasticSearch
---

### 一、原理简介

spring cloud bus整合java的事件处理机制和消息中间件的发送和接收，主要是由发送端、接收端和事件组成。
目前spring cloud bus只实现了RabbitMq和Kafka的封装。 但是Spring cloud Alibaba实现了对RocketMQ的封装。

spring cloud bus与spring cloud config的整合，并以RocketMQ作为消息代理。实现了应用配置的动态更新。

![消息总线](/images/posts/frameworks/springcloudbus.png)

向service A的实例3发送post请求，访问/bus/refresh接口，此时，service A的实例3就会将刷新请求发送到消息总线上，该消息事件会被service A的实例1和实例2从总线中获取到，并重新从config server中获取它们的配置信息，从而实现配置信息的动态更新。

### 二、 项目实战

#### 1. 准备工作
* 参考我的《Windows系统安装rocketMQ》安装rocketmq并且启动。

* 参考我的《Spring Cloud Config》创建ConfigServer和ConfigClient项目。



#### 2、改造config-client
修改POM文件增加spring-cloud-starter-bus-rocketmq依赖，和spring-boot-starter-actuator依赖。

完整的配置文件如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.5.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>vip.sunjin</groupId>
    <artifactId>ConfigClient</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.RELEASE</spring-cloud.version>
        <spring.cloud.alibaba.version>2.2.0.RELEASE</spring.cloud.alibaba.version>
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

            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring.cloud.alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>

        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-bus-rocketmq</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

    </dependencies>
</project>
```

在配置文件application.properties中加上rocketmq的配置。并需要加上spring.cloud.bus的三个配置，具体如下：

```properties
rocketmq.name-server=127.0.0.1:9876

spring.cloud.bus.enabled=true
spring.cloud.bus.trace.enabled=true
management.endpoints.web.exposure.include=bus-refresh
```

ConfigClientApplication启动类代码如下：

```java

@SpringBootApplication
@EnableDiscoveryClient
@RestController
@RefreshScope
public class ConfigClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigClientApplication.class, args);
    }

    @Value("${foo}")
    String foo;

    @RequestMapping(value = "/hi")
    public String hi(){
        return foo;
    }
}
```
依次启动confg-cserver,启动两个config-client，端口为：8881、8882。

访问http://localhost:8881/hi 或者http://localhost:8882/hi 浏览器显示：

```text
foo version 3
```

这时我们去代码仓库将foo的值改为“foo version 4”，即改变配置文件foo的值。如果是传统的做法，需要重启服务，才能达到配置文件的更新。
此时，我们只需要发送post请求：http://localhost:8881/actuator/bus-refresh， 你会发现config-client会重新读取配置文件。

```text
2021-03-27 13:43:19.900  INFO 2476 --- [nio-8881-exec-4] o.s.cloud.bus.event.RefreshListener      : Received remote refresh request. Keys refreshed [foo]
```

这时我们再访问http://localhost:8881/hi 或者http://localhost:8882/hi 浏览器显示：

```text
foo version 4
```
说明配置信息已经被更新了。

另外，/actuator/bus-refresh接口可以指定服务，即使用”destination”参数，比如 “/actuator/bus-refresh?destination=customers:**” 即刷新服务名为customers的所有服务。