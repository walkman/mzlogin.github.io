---
layout: post
title: Windows系统安装rocketMQ
categories: rocketMQ
excerpt: 为了学习消息中间件，学习spring cloud stream, 首先需要在Windows系统安装rocketMQ服务器和控制台。
tags: Windows, rocketMQ
---

### 一、预备环境
1. 系统

    Windows

2. 环境

    JDK1.8、Maven、Git
    
### 二. RocketMQ部署

1. 下载

   * 地址：https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.8.0/rocketmq-all-4.8.0-bin-release.zip
   * 解压已下载工程
   
2. 配置

   * 系统环境变量配置
   
   变量名：ROCKETMQ_HOME
   
   变量值：就是RocketMQ的解压路径， C:\Software\rocketmq-all-4.8.0-bin-release

3. 启动
   
   * 启动NAMESERVER
   
   Cmd命令框执行进入至‘ROCKETMQ_HOME\bin’下，然后执行‘start mqnamesrv.cmd’，启动NAMESERVER。成功后会弹出提示框，此框勿关闭。
   
   * 启动BROKER
   
   Cmd命令框执行进入至‘ROCKETMQ_HOME\bin’下，然后执行‘start mqbroker.cmd -n 127.0.0.1:9876 autoCreateTopicEnable=true’，启动BROKER。成功后会弹出提示框，此框勿关闭。
   
### 三. RocketMQ Console插件部署

1. 下载源代码

    执行：git clone https://github.com/apache/rocketmq-externals.git
    
2. 配置application.properties
    
    下载完成之后，进入‘rocketmq-externals\rocketmq-console\src\main\resources’文件夹，打开‘application.properties’进行配置。
    
```properties
server.address=127.0.0.1
server.port=28080
rocketmq.config.namesrvAddr=127.0.0.1:9876
```

3. 打包

   进入‘\rocketmq-externals\rocketmq-console’文件夹，执行‘mvn clean package -Dmaven.test.skip=true’，进行打包。


4. 启动Console

   打包成功之后，Cmd进入‘target’文件夹，执行‘java -jar rocketmq-console-ng-2.0.0.jar’，启动Console控制台。
   
5. 测试

   浏览器中打开 http://localhost:28080/
