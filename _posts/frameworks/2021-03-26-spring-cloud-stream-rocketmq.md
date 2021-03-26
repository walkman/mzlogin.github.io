---
layout: post
title: Spring Cloud Stream集成rocketMQ
categories: rocketMQ
excerpt: Spring Cloud Stream集成rocketMQ
tags: SpringCloud, rocketMQ
---

### 一、Spring Cloud Stream 介绍
Spring Cloud Stream 是一个用于构建基于消息的微服务应用框架，使用 Spring Integration 与 Broker 进行连接。
```text
一般来说，消息队列中间件都有一个 Broker Server（代理服务器），消息中转角色，负责存储消息、转发消息。
例如说在 RocketMQ 中，Broker 负责接收从生产者发送来的消息并存储、同时为消费者的拉取请求作准备。
另外，Broker 也存储消息相关的元数据，包括消费者组、消费进度偏移和主题和队列消息等。
```
Spring Cloud Stream 提供了消息中间件的统一抽象，推出了 publish-subscribe、consumer groups、partition 这些统一的概念。

Spring Cloud Stream 内部有两个概念：Binder 和 Binding。

##### Binder
跟消息中间件集成的组件，用来创建对应的 Binding。各消息中间件都有自己的 Binder 具体实现。
* Kafka 实现了 KafkaMessageChannelBinder
* RabbitMQ 实现了 RabbitMessageChannelBinder
* RocketMQ 实现了 RocketMQMessageChannelBinder

##### Binding
包括 Input Binding 和 Output Binding。Binding 在消息中间件与应用程序提供的 Provider 和 Consumer 之间提供了一个桥梁，
实现了开发者只需使用应用程序的 Provider 或 Consumer 生产或消费数据即可，屏蔽了开发者与底层消息中间件的接触。




### 二、用IDEA创建一个maven项目

#### 1. 创建依赖配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.3.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>
    <groupId>vip.sunjin</groupId>
    <artifactId>SpringCloudStream</artifactId>
    <version>1.0-SNAPSHOT</version>


    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Greenwich.RELEASE</spring-cloud.version>
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

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream-binder-rocketmq</artifactId>
            <version>0.2.2.RELEASE</version>
        </dependency>
    </dependencies>

</project>
```

#### 2. 创建属性配置application.yml

```properties
server:
  port: 9191

spring:
  application:
    name: spring-cloud-stream
  cloud:
    stream:
      rocketmq:
        binder:
          namesrv-addr: 127.0.0.1:9876
      bindings:
        input:
          content-type: text/plain
          destination: test-topic
          group: test-group1
        output:
          content-type: text/plain
          destination: test-topic
          group: demo-group
          
```
rocketmq服务需要提前安装好并且启动


#### 3. 创建启动类 并且增加发送消息的生产者

```java
@SpringBootApplication
@EnableBinding({ Source.class, Sink.class }) // 1
public class SendAndReceiveApplication {

    public static void main(String[] args) {
        SpringApplication.run(SendAndReceiveApplication.class, args);
    }

    @Bean // 2
    public CustomRunner customRunner() {
        return new CustomRunner();
    }

    public static class CustomRunner implements CommandLineRunner {

        @Autowired
        private Source source;

        @Override
        public void run(String... args) throws Exception {
            int count = 5;
            for (int index = 1; index <= count; index++) {
                source.output().send(MessageBuilder.withPayload("msg-" + index).build()); // 3
            }
        }
    }
}
```

#### 4. 创建一个消息消费者服务

```java
@Service
public class StreamListenerReceiveService {
    @StreamListener(Sink.INPUT) // 4
    public void receiveByStreamListener1(String receiveMsg) {
        System.out.println("receiveByStreamListener: " + receiveMsg);
    }
}
```

#### 5. 测试
启动项目就可以看到控制台输出日志
```text
receiveByStreamListener: msg-1
receiveByStreamListener: msg-2
receiveByStreamListener: msg-5
receiveByStreamListener: msg-3
receiveByStreamListener: msg-4
```

说明消息发送和接受都已经成功了。
