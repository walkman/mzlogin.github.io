---
layout: post
title: SpringCloud使用sleuth+zipkin实现链路追踪
categories: [SpringCloud,Sleuth]
excerpt: Spring Cloud Sleuth 主要功能就是在分布式系统中提供追踪解决方案，并且兼容支持了 zipkin，你只需要在pom文件中引入相应的依赖即可。
keywords: Sleuth
---

### 一、简介
微服务架构上通过业务来划分服务的，通过REST调用，对外暴露的一个接口，可能需要很多个服务协同才能完成这个接口功能，
如果链路上任何一个服务出现问题或者网络超时，都会形成导致接口调用失败。随着业务的不断扩张，服务之间互相调用会越来越复杂。
随着服务的越来越多，对调用链的分析会越来越复杂。所以需要使用Sleuth进行服务调用链路追踪。


### 二、项目实战

#### 1 构建server-zipkin
在spring Cloud为F版本的时候，已经不需要自己构建Zipkin Server了，只需要下载jar即可，

下载地址：
https://dl.bintray.com/openzipkin/maven/io/zipkin/java/zipkin-server/

当然也可以自己构建一个Zipkin服务。

这里我们直接下载zipkin-server-2.12.9-exec.jar 包并且启动jar包

```text
java -jar zipkin-server-2.12.9-exec.jar 
```

访问浏览器localhost:9411

#### 2 创建service-hi

在其pom引入起步依赖spring-cloud-starter-zipkin，代码如下：

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
    <artifactId>ServiceZipkin</artifactId>
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
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>

    </dependencies>

</project>
```

在其配置文件application.properties 指定zipkin server的地址，通过配置“spring.zipkin.base-url”指定：

```properties
server.port=8080
spring.zipkin.base-url=http://localhost:9411
spring.application.name=service-hi
```

通过引入spring-cloud-starter-zipkin依赖和设置spring.zipkin.base-url就可以了。


创建启动类并且对外暴露接口：
```java
@SpringBootApplication
@RestController
public class ServiceHiApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceHiApplication.class, args);
    }

    private static final Logger LOG = Logger.getLogger(ServiceHiApplication.class.getName());


    @Autowired
    private RestTemplate restTemplate;

    @Bean
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }

    @RequestMapping("/hi")
    public String callHome(){
        LOG.log(Level.INFO, "calling trace service-hi  ");
        return restTemplate.getForObject("http://localhost:8081/miya", String.class);
    }
    @RequestMapping("/info")
    public String info(){
        LOG.log(Level.INFO, "calling trace service-hi ");

        return "i'm service-hi";

    }

    @Bean
    public Sampler defaultSampler() {
        return Sampler.ALWAYS_SAMPLE;
    }
}

```

Sampler.ALWAYS_SAMPLE 是Zipkin的采样配置，没有这个配置Zipkin是无法采集链路追踪信息的。

#### 3. 创建service-miya

创建过程和service-hi相同，引入相同的依赖，配置下spring.zipkin.base-url， 修改端口号。

application.properties配置如下：
```properties
server.port=8081
spring.zipkin.base-url=http://localhost:9411
spring.application.name=service-hi
```

创建启动类并且对外暴露接口：

```java

@SpringBootApplication
@RestController
public class ServiceMiyaApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceMiyaApplication.class, args);
    }

    private static final Logger LOG = Logger.getLogger(ServiceMiyaApplication.class.getName());


    @RequestMapping("/hi")
    public String home(){
        LOG.log(Level.INFO, "hi is being called");
        return "hi i'm miya!";
    }

    @RequestMapping("/miya")
    public String info(){
        LOG.log(Level.INFO, "info is being called");
        return restTemplate.getForObject("http://localhost:8080/info",String.class);
    }

    @Autowired
    private RestTemplate restTemplate;

    @Bean
    public RestTemplate getRestTemplate(){
        return new RestTemplate();
    }

    @Bean
    public Sampler defaultSampler() {
        return Sampler.ALWAYS_SAMPLE;
    }
}
```

#### 4. 启动工程，演示追踪

依次启动上面的工程，打开浏览器访问：http://localhost:9411/, 就会看到Zipkin的控制台页面。

访问：http://localhost:8081/miya，浏览器出现：
```text
i'm service-hi
```

再打开http://localhost:9411/的界面，点击“依赖”,可以发现服务的依赖关系：
![Zipkin依赖](/images/posts/frameworks/zipkin-d.png)

点击“查找”,可以看到具体服务相互调用的数据：
![Zipkin查找](/images/posts/frameworks/zipkin-f.png)

可以看到Zipkin成功实现了服务链路追踪。


