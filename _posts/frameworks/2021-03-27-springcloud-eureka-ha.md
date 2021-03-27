---
layout: post
title: SpringCloud 高可用的服务注册中心
categories: [SpringCloud,Eureka]
excerpt: Eureka通过运行多个实例，使其更具有高可用性。事实上，这是它默认的熟性，你需要做的就是给对等的实例一个合法的关联serviceurl。
keywords: Eureka
---
### 一、简介
Eureka通过运行多个实例，使其更具有高可用性。事实上，这是它默认的熟性，你需要做的就是给对等的实例一个合法的关联serviceurl。

### 二、 项目实战

#### 创建一个maven项目EurekaServer

##### 1. 添加pom依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.walkersun</groupId>
    <artifactId>EurekaServer</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.5.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-dependencies -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.RELEASE</version>
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

        <!--eureka server -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>

        <!-- spring boot test-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

后面要打包项目，这里一定要配置maven插件。

##### 2. 添加属性配置文件

application-peer1.yml
```properties
server:
  port: 8761

spring:
  application:
    name: eureka-server1

eureka:
  instance:
    hostname: peer1
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://peer2:8762/eureka/

```

application-peer2.yml
```properties
server:
  port: 8762

spring:
  application:
    name: eureka-server2

eureka:
  instance:
    hostname: peer2
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://peer1:8761/eureka/

```

##### 3. 修改域名解析，
在C:/Windows/System32/drivers/etc/hosts 修改。

```text
127.0.0.1 peer1
127.0.0.1 peer2
```

##### 4. 创建启动类
```
@EnableEurekaServer
@SpringBootApplication
public class EurekaserverApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaserverApplication.class, args);
    }

}
```

##### 5. 打包并且运行注册中心

```text
java -jar EurekaServer-1.0-SNAPSHOT.jar --spring.profiles.active=peer1
java -jar EurekaServer-1.0-SNAPSHOT.jar --spring.profiles.active=peer2
```

#### 创建EurekaClient项目

##### 1. 添加pom依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.wakersun</groupId>
    <artifactId>EurekaProvider</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.5.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-dependencies -->
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.RELEASE</version>
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

        <!-- https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-starter-netflix-eureka-client -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>


    </dependencies>


    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

</project>
```

##### 2. 添加属性配置文件
application.yml

```properties
server:
  port: 9090

spring:
  application:
    name: service-hi

eureka:
  client:
    service-url:
      default-zone: http://peer1:8761/eureka/

```
##### 3. 创建启动类
```java
@SpringBootApplication
@EnableEurekaClient
public class ServiceHiApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceHiApplication.class, args);
    }

}
```

##### 4.启动EurekaClient微服务

访问：localhost:8761,如图：
![高可用注册中心](/images/posts/frameworks/eureka-ha.png)

你会发现注册了service-hi，并且有个peer2节点，同理访问localhost:8762你会发现有个peer1节点。
client只向8761注册，但是你打开8762，你也会发现，8762也有 service-hi的注册信息。

Eureka-eserver peer1 8761,Eureka-eserver peer2 8762相互感应，当有服务注册时，两个Eureka-eserver是对等的，
它们都存有相同的信息，这就是通过服务器的冗余来增加可靠性，当有一台服务器宕机了，服务并不会终止，因为另一台服务存有相同的数据。