---
layout: post
title: ElasticSearch 入门
categories: [frameworks,内功心法]
excerpt: Elasticsearch是一个实时分布式搜索和分析引擎。它让你以前所未有的速度处理大数据成为可能。
         它用于全文搜索、结构化搜索、分析以及将这三者混合使用。
keywords: ElasticSearch
---


## 1.ElasticSearch是什么
Elasticsearch是一个实时分布式搜索和分析引擎。它让你以前所未有的速度处理大数据成为可能。
它用于全文搜索、结构化搜索、分析以及将这三者混合使用。

## 2.ElasticSearch下载安装
本文以windows7 64位系统 演示。
### 2.1 去官方网站下载zip包解压到指定目录就完成安装了
[https://www.elastic.co/downloads/elasticsearch](https://www.elastic.co/downloads/elasticsearch)

### 2.2 启动ElasticSearch
1. 启动之前需要安装JDK8以及以上版本，并且设置JAVA_HOME环境变量。
2. 命令行进入ElasticSearch安装目录，执行 *bin\elasticsearch* 启动。
3. 如果启动失败可能是ElasticSearch默认的内存设置比较大，你的机器内存不够， 去安装目录中修改配置项 *config\jvm.options* 然后重新启动。

### 2.3 访问 http://localhost:9200/
页面上看到如下信息：
```json
{
  "name" : "JicjuEw",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "G487MaJ5TkisDPVHHF1DcA",
  "version" : {
    "number" : "6.3.1",
    "build_flavor" : "default",
    "build_type" : "zip",
    "build_hash" : "eb782d0",
    "build_date" : "2018-06-29T21:59:26.107521Z",
    "build_snapshot" : false,
    "lucene_version" : "7.3.1",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
说明启动成功

## 3. 安装插件
想要测试增删改查的功能，还需要安装一个插件，用来在页面执行ElasticSearch的交互操作。

### 3.1 下载安装NodeJS
主要是NodeJs包含了npm工具，安装插件需要使用这个工具。下操作系统对应的版本。 本文使用 *node-v8.11.4-x64.msi*
[https://nodejs.org/en/download/](https://nodejs.org/en/download/)

### 3.2 下载插件
*git clone https://github.com/mobz/elasticsearch-head.git*

### 3.3 安装插件
进入elasticsearch-head项目目录，执行安装命令 *npm install*

### 3.4 启动
*npm run start*

### 3.5 访问插件
[http://localhost:9100/](http://localhost:9100/)
如果能正常打开说明插件安装成功了。
现在集群健康状态哪里显示未连接，这是因为head插件没有权限获取集群节点的信息，接下来设置权限

## 4. 设置权限
进入config目录，打开配置文件 *elasticsearch.yml*，添加配置项。
```yaml
# elasticsearch中启用CORS
http.cors.enabled: true
# 允许访问的IP地址段，* 为所有IP都可以访问
http.cors.allow-origin: "*"
```
重新启动elasticsearch，再次访问[http://localhost:9100/](http://localhost:9100/) 发现已经连接成功。

现在就可以执行增删改查操作了。