---
layout: post
title: gRPC 非官方教程
categories: gRPC
excerpt: gRPC是一个高性能、开源和通用的 RPC 框架，支持多语言。gRPC 基于 HTTP/2 标准设计，带来诸如双向流、流控、头部压缩、单 TCP 连接上的多复用请求等特性。这些特性使得其在移动设备上表现更好，更省电和节省空间占用。
tags: gRPC
---

### 一、 简介

gRPC的定义：

1. 一个高性能、通用的开源RPC框架
2. 主要面向移动应用开发： gRPC提供了一种简单的方法来精确地定义服务和为iOS、Android和后台支持服务自动生成可靠性很强的客户端功能库。
3. 基于HTTP/2协议标准而设计，基于ProtoBuf(Protocol Buffers)序列化协议开发
4. 支持众多开发语言


![grpc架构图](/images/posts/frameworks/grpc_concept_diagram_00.png)


### 二、 项目实战Hello World

主要流程：
1. 创建maven项目
2. 添加grpc依赖,protobuf依赖和插件
3. 通过.proto文件定义服务
4. 通过protocol buffer compiler插件生成客户端和服务端
5. 通过grpc API生成客户端和服务端代码

#### 1. 创建maven项目
添加pom依赖, 包含依赖包和生成基础类的插件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>vip.sunjin</groupId>
    <artifactId>GrpcServer</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <grpc.version>1.36.1</grpc.version>
        <protobuf.version>3.15.6</protobuf.version>
    </properties>


    <dependencies>

        <!-- protobuf -->
        <dependency>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
            <version>${protobuf.version}</version>
        </dependency>

        <!-- GRPC -->
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-netty</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-protobuf</artifactId>
            <version>${grpc.version}</version>
        </dependency>
        <dependency>
            <groupId>io.grpc</groupId>
            <artifactId>grpc-stub</artifactId>
            <version>${grpc.version}</version>
        </dependency>

    </dependencies>
    <build>
        <extensions>
            <extension>
                <groupId>kr.motd.maven</groupId>
                <artifactId>os-maven-plugin</artifactId>
                <version>1.6.2</version>
            </extension>
        </extensions>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.xolstice.maven.plugins</groupId>
                <artifactId>protobuf-maven-plugin</artifactId>
                <version>0.6.1</version>
                <configuration>
                    <protocArtifact>com.google.protobuf:protoc:3.12.0:exe:${os.detected.classifier}</protocArtifact>
                    <pluginId>grpc-java</pluginId>
                    <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.36.0:exe:${os.detected.classifier}</pluginArtifact>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>compile-custom</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

</project>
```

#### 2. 定义RPC服务数据结构 proto文件

创建一个文件夹src/main/proto/

创建一个helloworld.proto文件

```text

syntax = "proto3";

option java_multiple_files = true;
option java_package = "vip.sunjin.examples.helloworld";
option java_outer_classname = "HelloWorldProto";


package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}

```


#### 3. 生成基础类
使用maven编译项目 生成基础类

```text
mvn compile
```

生成后的类文件如下：

```yaml
target
  classes
    vip
      sunjin
        examples
          helloworld
            GreeterGrpc.java
            HelloReply.java
            HelloReplyOrBuilder.java
            HelloRequest.java
            HelloRequestOrBuilder.java
            HelloWorldClient.java
            HelloWorldProto.java
            HelloWorldServer.java
```

GreeterGrpc封装基本的GRPC功能，后续的客户端和服务端都从这个类引申出来。

#### 4. 创建服务端

```java
/**
 * Server that manages startup/shutdown of a {@code Greeter} server.
 */
public class HelloWorldServer {
  private static final Logger logger = Logger.getLogger(HelloWorldServer.class.getName());

  private Server server;

  private void start() throws IOException {
    /* The port on which the server should run */
    int port = 50051;
    server = ServerBuilder.forPort(port)
        .addService(new GreeterImpl())
        .build()
        .start();
    logger.info("Server started, listening on " + port);
    Runtime.getRuntime().addShutdownHook(new Thread() {
      @Override
      public void run() {
        // Use stderr here since the logger may have been reset by its JVM shutdown hook.
        System.err.println("*** shutting down gRPC server since JVM is shutting down");
        try {
          HelloWorldServer.this.stop();
        } catch (InterruptedException e) {
          e.printStackTrace(System.err);
        }
        System.err.println("*** server shut down");
      }
    });
  }

  private void stop() throws InterruptedException {
    if (server != null) {
      server.shutdown().awaitTermination(30, TimeUnit.SECONDS);
    }
  }

  /**
   * Await termination on the main thread since the grpc library uses daemon threads.
   */
  private void blockUntilShutdown() throws InterruptedException {
    if (server != null) {
      server.awaitTermination();
    }
  }

  /**
   * Main launches the server from the command line.
   */
  public static void main(String[] args) throws IOException, InterruptedException {
    final HelloWorldServer server = new HelloWorldServer();
    server.start();
    server.blockUntilShutdown();
  }

  static class GreeterImpl extends GreeterGrpc.GreeterImplBase {

    @Override
    public void sayHello(HelloRequest req, StreamObserver<HelloReply> responseObserver) {
      HelloReply reply = HelloReply.newBuilder().setMessage("Hello " + req.getName()).build();
      responseObserver.onNext(reply);
      responseObserver.onCompleted();
    }
  }
}
```
服务端只需要指定一个端口号，然后暴露一个服务。


#### 5. 创建客户端

```java
public class HelloWorldClient {
  private static final Logger logger = Logger.getLogger(HelloWorldClient.class.getName());

  private final GreeterGrpc.GreeterBlockingStub blockingStub;

  /** Construct client for accessing HelloWorld server using the existing channel. */
  public HelloWorldClient(Channel channel) {
    // 'channel' here is a Channel, not a ManagedChannel, so it is not this code's responsibility to
    // shut it down.

    // Passing Channels to code makes code easier to test and makes it easier to reuse Channels.
    blockingStub = GreeterGrpc.newBlockingStub(channel);
  }

  /** Say hello to server. */
  public void greet(String name) {
    logger.info("Will try to greet " + name + " ...");
    HelloRequest request = HelloRequest.newBuilder().setName(name).build();
    HelloReply response;
    try {
      response = blockingStub.sayHello(request);
    } catch (StatusRuntimeException e) {
      logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
      return;
    }
    logger.info("Greeting Reply: " + response.getMessage());
  }

  /**
   * Greet server. If provided, the first element of {@code args} is the name to use in the
   * greeting. The second argument is the target server.
   */
  public static void main(String[] args) throws Exception {
    String user = "neil";
    // Access a service running on the local machine on port 50051
    String target = "localhost:50051";


    // Create a communication channel to the server, known as a Channel. Channels are thread-safe
    // and reusable. It is common to create channels at the beginning of your application and reuse
    // them until the application shuts down.
    ManagedChannel channel = ManagedChannelBuilder.forTarget(target)
        // Channels are secure by default (via SSL/TLS). For the example we disable TLS to avoid
        // needing certificates.
        .usePlaintext()
        .build();
    try {
      HelloWorldClient client = new HelloWorldClient(channel);
      client.greet(user);
    } finally {
      // ManagedChannels use resources like threads and TCP connections. To prevent leaking these
      // resources the channel should be shut down when it will no longer be used. If it may be used
      // again leave it running.
      channel.shutdownNow().awaitTermination(5, TimeUnit.SECONDS);
    }
  }
}
```

客户端需要指定调用服务的地址和端口号并且通过调用桩代码调用服务端的服务。

客户端和服务端是直连的。


#### 6. 测试
先启动服务端代码 HelloWorldServer

然后执行客户端代码 HelloWorldClient

执行结果如下：

```text
四月 02, 2021 4:54:40 下午 vip.sunjin.examples.helloworld.HelloWorldClient greet
信息: Will try to greet neil ...
四月 02, 2021 4:54:41 下午 vip.sunjin.examples.helloworld.HelloWorldClient greet
信息: Greeting Reply: Hello neil
```

### 三、 grpc服务端流
一般业务场景下，我们都是使用grpc的simple-rpc模式，也就是每次客户端发起请求，服务端会返回一个响应结果的模式。

但是grpc除了这种一来一往的请求模式外，还有流式模式。

服务端流模式是说客户端发起一次请求后，服务端在接受到请求后，可以以流的方式，使用同一连接，不断的向客户端写回响应结果，客户端则可以源源不断的接受到服务端写回的数据。

下面我们通过简单例子，来说明如何使用，服务端端流。

#### 1. 定义RPC服务数据结构 proto文件

MetricsService.proto
```text
syntax = "proto3";

option java_multiple_files = true;
option java_package = "vip.sunjin.examples.helloworld";
option java_outer_classname = "MetricsServiceProto";


message Metric {
  int64 metric = 2;
}

message Average {
  double val = 1;
}

service MetricsService {
  rpc collectServerStream (Metric) returns (stream Average);
}
```
然后使用maven编译项目 生成基础类

#### 2.创建服务端代码

服务实现类
```java
public class MetricsServiceImpl extends MetricsServiceGrpc.MetricsServiceImplBase {
    private static final Logger logger = Logger.getLogger(MetricsServiceImpl.class.getName());
    /**
     * 服务端流
     * @param request
     * @param responseObserver
     */
    @Override
    public void collectServerStream(Metric request, StreamObserver<Average> responseObserver) {
        logger.info("received request : " +  request.getMetric());
        for(int i = 0; i  < 10; i++){
            responseObserver.onNext(Average.newBuilder()
                    .setVal(new Random(1000).nextDouble())
                    .build());
            logger.info("send to client");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        responseObserver.onCompleted();
    }

}
```

服务端Server启动
```java
public class MetricsServer {
    private static final Logger logger = Logger.getLogger(MetricsServer.class.getName());
    public static void main(String[] args) throws IOException, InterruptedException {
        int port = 50051;
//        //启动服务
        MetricsServiceImpl metricsService = new MetricsServiceImpl();
        Server server = ServerBuilder.forPort(port).addService(metricsService).build();
        server.start();

        logger.info("Server started, listening on " + port);

        server.awaitTermination();
    }
}
```


#### 3.创建客户端代码

通过BlockingStub 调用服务
```java
public class MetricsClient {
    private static final Logger logger = Logger.getLogger(MetricsClient.class.getName());

    public static void main(String[] args) {
        int port = 50051;
        //获取客户端桩对象
        ManagedChannel channel = ManagedChannelBuilder.forTarget("localhost:" + port).usePlaintext().build();
        logger.info("new channel");
        MetricsServiceGrpc.MetricsServiceBlockingStub stub = MetricsServiceGrpc.newBlockingStub(channel);
        logger.info("new stub");
        //发起rpc请求，设置StreamObserver用于监听服务器返回结果
        Iterator<Average> iterator = stub.collectServerStream(Metric.newBuilder().setMetric(1L).build());

        while (iterator.hasNext()){
            logger.info("call result: " + iterator.next().getVal());
        }
    }
}
```

#### 4.测试
先启动服务端，
再启动客户端后，可以看到iterator会源源不断的接受到服务端返回的数据。

```text
四月 03, 2021 8:44:56 下午 vip.sunjin.examples.helloworld.MetricsClient main
信息: call result: 0.7101849056320707
四月 03, 2021 8:44:57 下午 vip.sunjin.examples.helloworld.MetricsClient main
信息: call result: 0.7101849056320707
四月 03, 2021 8:44:58 下午 vip.sunjin.examples.helloworld.MetricsClient main
信息: call result: 0.7101849056320707
四月 03, 2021 8:44:59 下午 vip.sunjin.examples.helloworld.MetricsClient main
信息: call result: 0.7101849056320707
```

服务端流使用场景：

* 客户端请求一次，但是需要服务端源源不断的返回大量数据时候，比如大批量数据查询的场景。
* 比如客户端订阅服务端的一个服务数据，服务端发现有新数据时，源源不断的吧数据推送给客户端。

### 三、 grpc客户端流

客户端流模式是说客户端发起请求与服务端建立链接后，可以使用同一连接，不断的向服务端传送数据，等客户端把全部数据都传送完毕后，服务端才返回一个请求结果。

