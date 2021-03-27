---
layout: post
title: Springboot快速集成rocketMQ
categories: [Springboot,RocketMQ]
excerpt: RocketMQ是目前主流的消息中间件之一，并且自身就支持分布式功能。最初由阿里巴巴团队开发，并且经历过双十一等海量消息场景的考验，后捐赠给Apache开源基金会，这也是为什么我们经常听说RocketMQ是阿里巴巴的消息中间件，项目却在Apache的顶级项目中。
tags: Springboot, rocketMQ
---

### 一、依赖集成
首先创建一个SpringBoot项目，为了方便通过浏览器访问测试，引入web对应的starter。

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
    <artifactId>SpringBootRocketMQ</artifactId>
    <version>1.0-SNAPSHOT</version>
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-spring-boot-starter</artifactId>
            <version>2.1.1</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>

</project>

```

### 二、配置文件
application.yml

```properties
server:
  port: 9292

spring:
  application:
    name: spring-boot-rocketmq


rocketmq:
  name-server: 127.0.0.1:9876
  producer:
    group: demo
  demo:
    topic: test-topic-1
    tag:
      register: register
      modify: modify
    group:
      register: register
      modify: modify
```


### 三、 对RocketMQTemplate进行封装

在项目中每次使用都注入一个RocketMQTemplate并不符合面向对象的思想，而且RocketMQTemplate还提供了多个常用的方法，
比如同步、异步、直接发送等模式。我们可以将其封装成为一个通用的Service，这样其他服务只需注入对应的Service，
调用公共的方法即可，并且注明每个方法的使用场景。
抽象出来的Service接口如下：
```java
/**
 * Rocket MQ 对应服务封装
 *
 **/
public interface RocketMqService {

    /**
     * 同步发送消息<br/>
     * <p>
     * 当发送的消息很重要是，且对响应时间不敏感的时候采用sync方式;
     *
     * @param mqMsg 发送消息实体类
     */
    void send(MqMsg mqMsg);

    /**
     * 异步发送消息，异步返回消息结果<br/>
     * <p>
     * 当发送的消息很重要，且对响应时间非常敏感的时候采用async方式；
     *
     * @param mqMsg 发送消息实体类
     */
    void asyncSend(MqMsg mqMsg);

    /**
     * 直接发送发送消息，不关心返回结果，容易消息丢失，适合日志收集、不精确统计等消息发送;<br/>
     * <p>
     * 当发送的消息不重要时，采用one-way方式，以提高吞吐量；
     *
     * @param mqMsg 发送消息实体类
     */
    void sendOneWay(MqMsg mqMsg);

}
```
定义了不同类型消息发送的方法，同时在注释部分说明具体方法的使用场景。其中将发送的参数封装为MqMsg对象，MqMsg的结构如下：

```java

import lombok.Getter;
import lombok.Setter;

@Getter
@Setter
public class MqMsg {

    /**
     * 一级消息：消息topic
     */
    private String topic;

    /**
     * 二级消息：消息topic对应的tags
     */
    private String tags;

    /**
     * 消息内容
     */
    private String content;

}
```

其中，topic为消息的主题，content为消息的内容，具体内容可根据生产者和消费者之间进行协定。

针对上述的接口，提供具体的方法实现：

```java
import org.apache.rocketmq.client.producer.SendCallback;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.spring.core.RocketMQTemplate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Service;

@Service("rocketMqService")
public class RocketMqServiceImpl implements RocketMqService {

    private static final Logger log = LoggerFactory.getLogger(RocketMqServiceImpl.class);

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    @Override
    public void send(MqMsg mqMsg) {
        log.info("send发送消息到mqMsg={}", mqMsg);
        rocketMQTemplate.send(mqMsg.getTopic() + ":" + mqMsg.getTags(),
                MessageBuilder.withPayload(mqMsg.getContent()).build());
    }

    @Override
    public void asyncSend(MqMsg mqMsg) {
        log.info("asyncSend发送消息到mqMsg={}", mqMsg);
        rocketMQTemplate.asyncSend(mqMsg.getTopic() + ":" + mqMsg.getTags(), mqMsg.getContent(),
                new SendCallback() {
                    @Override
                    public void onSuccess(SendResult sendResult) {
                        // 成功不做日志记录或处理
                    }

                    @Override
                    public void onException(Throwable throwable) {
                        log.info("mqMsg={}消息发送失败", mqMsg);
                    }
                });
    }

    @Override
    public void sendOneWay(MqMsg mqMsg) {
        log.info("syncSendOrderly发送消息到mqMsg={}", mqMsg);
        rocketMQTemplate.sendOneWay(mqMsg.getTopic() + ":" + mqMsg.getTags(), mqMsg.getContent());
    }
}
```
其中异步发送方法asyncSend的异步返回结果中可以根据具体的业务场景进行针对性的处理。

上述方法中均默认使用了tag对topic进行分类。如果具体的业务中不需要tag，则可对上述方法中拼接的冒号+tag部分去除。本实例使用tag进行分类，方便用到时可以借鉴。

完成了上面的封装之后，在业务使用场景中只需注入对应的Service即可，后面测试时我们会进行演示。


### 四、消费者示例
关于消费者我们可以直接实现RocketMQListener接口，然后通过@RocketMQMessageListener注解来匹配目标消息。

这里为了演示统一topic下不同的tag的使用方法，分两个消费者来进行演示，直接看代码：

```java
import org.apache.rocketmq.spring.annotation.RocketMQMessageListener;
import org.apache.rocketmq.spring.core.RocketMQListener;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

/**
 * 消息队列消费端使用示例
 *
 **/
@Service
@RocketMQMessageListener(topic = "${rocketmq.demo.topic}"
        , consumerGroup = "${rocketmq.demo.group.register}"
        , selectorExpression = "${rocketmq.demo.tag.register}")
public class MqRegisteredListener implements RocketMQListener<String> {

    private static final Logger log = LoggerFactory.getLogger(MqRegisteredListener.class);

    @Override
    public void onMessage(String message) {
        log.info("received registered message: {}", message);
    }
}

```

下面再来看看第二个监听“更新”功能的tag代码实现：

```java

import org.apache.rocketmq.spring.annotation.RocketMQMessageListener;
import org.apache.rocketmq.spring.core.RocketMQListener;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

/**
 * 消息队列消费端使用示例
 *
 **/
@Service
@RocketMQMessageListener(topic = "${rocketmq.demo.topic}"
        , consumerGroup = "${rocketmq.demo.group.modify}"
        , selectorExpression = "${rocketmq.demo.tag.modify}")
public class MqModifyListener implements RocketMQListener<String> {

    private static final Logger log = LoggerFactory.getLogger(MqRegisteredListener.class);

    @Override
    public void onMessage(String message) {
        log.info("received modify message: {}", message);
    }
}
```

这个消费者与第一个不同的地方有两个，均在注解部分。其中topic两个是相同的，selectorExpression一个针对注册功能的tag进行过滤，一个针对修改信息的功能进行过滤。而消费组consumerGroup对应的值不能相同，否则启动时会抛出异常。

经过上述步骤，已经完成了生产者、消费者的配置。下面就同一个测试来验证服务的执行情况。

### 五、测试验证

定义一个Controller，用于外部请求，触发用户注册和修改操作的消息发送。

```java

import org.springframework.beans.factory.annotation.Value;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import javax.annotation.Resource;
import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/demo")
public class DemoController {

    @Resource
    private RocketMqService rocketMqService;

    @Value("${rocketmq.demo.topic}")
    private String topic;
    @Value("${rocketmq.demo.tag.register}")
    private String tagRegister;
    @Value("${rocketmq.demo.tag.modify}")
    private String tagModify;


    @GetMapping("/send")
    public void send() {
        MqMsg mqMsg = new MqMsg();
        mqMsg.setTopic(topic);
        mqMsg.setTags(tagRegister);

        // 此处可为其他VO对象，替换掉Map
        Map<String, String> userInfo = new HashMap<>();
        userInfo.put("username", "zhangsan");
        userInfo.put("age", "12");
        // 此处可封装为json等格式
        mqMsg.setContent(userInfo.toString());
        // 第一个发送注册消息
        rocketMqService.asyncSend(mqMsg);

        mqMsg.setTags(tagModify);
        userInfo.put("age", "18");
        // 此处可封装为json等格式
        mqMsg.setContent(userInfo.toString());
        // 发送修改消息
        rocketMqService.asyncSend(mqMsg);
    }
}

```

最后还要有个启动类

```java

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AppStart {

    public static void main(String[] args) {
        SpringApplication.run(AppStart.class, args);
    }

}

```

在上述测试中第一部分发送了注册用户的消息，第二部分针对注册的消息进行了修改，又发送了一个消息。消息内容直接将Map转换为字符串了，在实战的过程中可根据双方协商，比如采用Json或其他序列化方法。

然后在浏览器访问：http://localhost:9292/demo/send ，触发消息的发送。

此时查看控制台，几乎在瞬间，就可以看到如下日志信息：

```text
2021-03-26 18:32:15.677  INFO 12044 --- [nio-9292-exec-1] v.s.s.rocketmq.RocketMqServiceImpl       : asyncSend发送消息到mqMsg=vip.sunjin.springboot.rocketmq.MqMsg@2754f4b8
2021-03-26 18:32:15.690  INFO 12044 --- [nio-9292-exec-1] v.s.s.rocketmq.RocketMqServiceImpl       : asyncSend发送消息到mqMsg=vip.sunjin.springboot.rocketmq.MqMsg@2754f4b8
2021-03-26 18:32:15.847  INFO 12044 --- [MessageThread_1] v.s.s.rocketmq.MqRegisteredListener      : received modify message: {age=18, username=zhangsan}
2021-03-26 18:32:15.847  INFO 12044 --- [MessageThread_1] v.s.s.rocketmq.MqRegisteredListener      : received registered message: {age=12, username=zhangsan}
```

很显然，消费者已经成功接收到消息，并且同一个topic，根据不同的tag分别进行处理。至此，一个完整是示例演示完毕。

### 六、小结

关于消息队列，还是其他很多方法都位于RocketMQTemplate当中，根据业务需要可以继续封装对应的Service。关于tag也还有不同的使用方式，大家可基于该文提供的基本框架和基本思路进一步完善。
