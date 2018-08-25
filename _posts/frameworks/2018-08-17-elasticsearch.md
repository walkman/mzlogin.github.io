---
layout: post
title: ElasticSearch 入门
categories: [frameworks,内功心法]
excerpt: Elasticsearch是一个实时分布式搜索和分析引擎。它让你以前所未有的速度处理大数据成为可能。
         它用于全文搜索、结构化搜索、分析以及将这三者混合使用。
keywords: ElasticSearch
---

### 一、ElasticSearch 基本概念
Elasticsearch是一个实时分布式搜索和分析引擎。它让你以前所未有的速度处理大数据成为可能。它用于全文搜索、结构化搜索、分析以及将这三者混合使用。

先看一个关系型数据库和ES的概念对比
```yaml
Relational DB -> Databases -> Tables -> Rows     -> Columns
Elasticsearch -> Indices   -> Types -> Documents -> Fields
```
Elasticsearch集群可以包含多个索引(indices)（数据库），每一个索引可以包含多个类型(types)（表），每一个类型包含多
个文档(documents)（行），然后每个文档包含多个字段(Fields)（列）。

#### 数据层面
* 索引（index）
ElasticSearch将它的数据存储在一个或多个索引（index）中。用SQL领域的术语来类比，索引就像数据库，可以向索引写入文档或者从索引中读取文档，
并通过ElasticSearch内部使用Lucene将数据写入索引或从索引中检索数据。
* 类型（type）
每个文档都有与之对应的类型（type）定义。这允许用户在一个索引中存储多种文档类型，并为不同文档提供类型提供不同的映射。
* 文档（document）
文档（document）是ElasticSearch中的主要实体。对所有使用ElasticSearch的案例来说，他们最终都可以归结为对文档的搜索。文档由字段构成。
* 映射（mapping）
所有文档写进索引之前都会先进行分析，如何将输入的文本分割为词条、哪些词条又会被过滤，这种行为叫做映射（mapping）。一般由用户自己定义规则。

#### 服务层面
* 接近实时（NRT）
Elasticsearch 是一个接近实时的搜索平台。这意味着，从索引一个文档直到这个文档能够被搜索到有一个很小的延迟（通常是 1 秒）。
* 集群（cluster）
代表一个集群，集群中有多个节点（node），其中有一个为主节点，这个主节点是可以通过选举产生的，主从节点是对于集群内部来说的。
es的一个概念就是去中心化，字面上理解就是无中心节点，这是对于集群外部来说的，因为从外部来看es集群，在逻辑上是个整体，
你与任何一个节点的通信和与整个es集群通信是等价的。
* 分片（shards）
代表索引分片，es可以把一个完整的索引分成多个分片，这样的好处是可以把一个大的索引拆分成多个，分布到不同的节点上。构成分布式搜索。分片的数量只能在索引创建前指定，并且索引创建后不能更改。5.X默认不能通过配置文件定义分片
* 副本（replicas）
代表索引副本，es可以设置多个索引的副本，副本的作用一是提高系统的容错性，当个某个节点某个分片损坏或丢失时可以从副本中恢复。二是提高es的查询效率，es会自动对搜索请求进行负载均衡。
* 数据恢复（recovery）
代表数据恢复或叫数据重新分布，es在有节点加入或退出时会根据机器的负载对索引分片进行重新分配，挂掉的节点重新启动时也会进行数据恢复。
GET /_cat/health?v   #可以看到集群状态
* 数据源（River）
代表es的一个数据源，也是其它存储方式（如：数据库）同步数据到es的一个方法。它是以插件方式存在的一个es服务，通过读取river中的数据并把它索引到es中，官方的river有couchDB的，RabbitMQ的，Twitter的，Wikipedia的，river这个功能将会在后面的文件中重点说到。
* 网关（gateway）
代表es索引的持久化存储方式，es默认是先把索引存放到内存中，当内存满了时再持久化到硬盘。当这个es集群关闭再重新启动时就会从gateway中读取索引数据。es支持多种类型的gateway，有本地文件系统（默认），分布式文件系统，Hadoop的HDFS和amazon的s3云存储服务。
* 自动发现（discovery.zen）
代表es的自动发现节点机制，es是一个基于p2p的系统，它先通过广播寻找存在的节点，再通过多播协议来进行节点之间的通信，同时也支持点对点的交互。
5.X关闭广播，需要自定义
* 通信（Transport）
代表es内部节点或集群与客户端的交互方式，默认内部是使用tcp协议进行交互，同时它支持http协议（json格式）、thrift、servlet、memcached、zeroMQ等的传输协议（通过插件方式集成）。
节点间通信端口默认：9300-9400
* 分片和复制（shards and replicas）
一个索引可以存储超出单个结点硬件限制的大量数据。比如，一个具有10亿文档的索引占据1TB的磁盘空间，而任一节点可能没有这样大的磁盘空间来存储或者单个节点处理搜索请求，响应会太慢。
为了解决这个问题，Elasticsearch提供了将索引划分成多片的能力，这些片叫做分片。当你创建一个索引的时候，你可以指定你想要的分片的数量。每个分片本身也是一个功能完善并且独立的“索引”，这个“索引” 可以被放置到集群中的任何节点上。

### 二、安装运行ElasticSearch

#### 1.ElasticSearch下载安装
本文以windows7 64位系统 演示。
* 去官方网站下载zip包解压到指定目录就完成安装了
[https://www.elastic.co/downloads/elasticsearch](https://www.elastic.co/downloads/elasticsearch)

* 启动ElasticSearch
1. 启动之前需要安装JDK8以及以上版本，并且设置JAVA_HOME环境变量。
2. 命令行进入ElasticSearch安装目录，执行 *bin\elasticsearch* 启动。
3. 如果启动失败可能是ElasticSearch默认的内存设置比较大，你的机器内存不够， 去安装目录中修改配置项 *config\jvm.options* 然后重新启动。

* 访问 http://localhost:9200/
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

#### 2. 安装插件
想要测试增删改查的功能，还需要安装一个插件，用来在页面执行ElasticSearch的交互操作。

* 下载安装NodeJS
主要是NodeJs包含了npm工具，安装插件需要使用这个工具。下操作系统对应的版本。 本文使用 *node-v8.11.4-x64.msi*
[https://nodejs.org/en/download/](https://nodejs.org/en/download/)

* 下载插件
*git clone https://github.com/mobz/elasticsearch-head.git*

* 安装插件
进入elasticsearch-head项目目录，执行安装命令 *npm install*

* 启动
*npm run start*

* 访问插件
[http://localhost:9100/](http://localhost:9100/)
如果能正常打开说明插件安装成功了。
现在集群健康状态哪里显示未连接，这是因为head插件没有权限获取集群节点的信息，接下来设置权限

#### 3.设置权限
进入config目录，打开配置文件 *elasticsearch.yml*，添加配置项。
```yaml
# elasticsearch中启用CORS
http.cors.enabled: true
# 允许访问的IP地址段，* 为所有IP都可以访问
http.cors.allow-origin: "*"
```
重新启动elasticsearch，再次访问[http://localhost:9100/](http://localhost:9100/) 发现已经连接成功。

现在就可以执行增删改查操作了。


### 三、实战练习
* **在索引中新建一个索引**

*megacorp*

#### 1.PUT

*PUT /megacorp/employee/1*
```json
{
    "first_name" : "John",
    "last_name" : "Smith",
    "age" : 25,
    "about" : "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```

![elasticsearch-put.png](/images/posts/frameworks/elasticsearch-put.png)
看到右边返回结果就是创建成功了

继续在增加两条数据

*PUT /megacorp/employee/2*
```json
{
    "first_name" : "Jane",
    "last_name" : "Smith",
    "age" : 32,
    "about" : "I like to collect rock albums",
    "interests": [ "music" ]
}
```
*PUT /megacorp/employee/3*
```json
{
    "first_name" : "Douglas",
    "last_name" : "Fir",
    "age" : 35,
    "about": "I like to build cabinets",
    "interests": [ "forestry" ]
}
```

#### 2.GET

*megacorp/employee/3*

返回结果：
```json
{
    "_index": "megacorp",
    "_type": "employee",
    "_id": "3",
    "_version": 1,
    "found": true,
    "_source": {
        "first_name": "Douglas",
        "last_name": "Fir",
        "age": 35,
        "about": "I like to build cabinets",
        "interests": [
            "forestry"
         ]
    }
}
```

我们通过HTTP方法 GET 来检索文档，同样的，我们可以使用 DELETE 方法删除文档，使用 HEAD 方法检查某文档是否存在。如果想更新已存在的文档，我们只需再 PUT 一次。

#### 3.SEARCH

*GET /megacorp/employee/_search?q=last_name:Smith*

我们在请求中使用 _search 关键字，然后将查询语句传递给参数 q= 。这样就可以得到所有姓氏为Smith的结果：

```json
{
    "hits": {
        "total": 2,
        "max_score": 0.30685282,
        "hits": [
            {
                "_source": {
                    "first_name": "John",
                    "last_name": "Smith",
                    "age": 25,
                    "about": "I love to go rock climbing",
                    "interests": [ "sports", "music" ]
                    }
            },
            {
                "_source": {
                    "first_name": "Jane",
                    "last_name": "Smith",
                    "age": 32,
                    "about": "I like to collect rock albums",
                    "interests": [ "music" ]
                    }
            }
        ]
    }
}
```

#### 4.DSL语句查询
查询字符串搜索便于通过命令行完成特定(ad hoc)的搜索，但是它也有局限性（参阅简单搜索章节）。
Elasticsearch提供丰富且灵活的查询语言叫做DSL查询(Query DSL),它允许你构建更加复杂、强大的查询。

*POST /megacorp/employee/_search*
```json
{
  "query": {
    "match": {
      "last_name": "Smith"
    }
  }
}
```

这会返回与之前查询相同的结果。

#### 5.更加复杂的查询
*POST /megacorp/employee/_search*
```json
{
  "query": {
    "bool": {
      "filter": {
        "range": {
          "age": {
            "gt": 30
          }
        }
      },
      "must": {
        "match": {
          "last_name": "Smith"
        }
      }
    }
  }
}
```

#### 6.全文搜索
```json
{
  "query": {
    "match": {
      "about": "rock climbing"
    }
  }
}
```
查询结果
```json
{
    "took":31,
    "timed_out":false,
    "_shards":{
        "total":5,
        "successful":5,
        "skipped":0,
        "failed":0
    },
    "hits":{
        "total":2,
        "max_score":0.5753642,
        "hits":[
            {
                "_index":"megacorp",
                "_type":"employee",
                "_id":"1",
                "_score":0.5753642,
                "_source":{
                    "first_name":"John",
                    "last_name":"Smith",
                    "age":25,
                    "about":"I love to go rock climbing",
                    "interests":[
                        "sports",
                        "music"
                    ]
                }
            },
            {
                "_index":"megacorp",
                "_type":"employee",
                "_id":"2",
                "_score":0.2876821,
                "_source":{
                    "first_name":"Jane",
                    "last_name":"Smith",
                    "age":32,
                    "about":"I like to collect rock albums",
                    "interests":[
                        "music"
                    ]
                }
            }
        ]
    }
}
```
"_score"字段是结果相关性评分。
默认情况下，Elasticsearch根据结果相关性评分来对结果集进行排序，所谓的「结果相关性评分」就是文档与查询条件的匹配程度。
这个例子很好的解释了Elasticsearch如何在各种文本字段中进行全文搜索，并且返回相关性最大的结果集。

#### 7.短语搜索
有时候你想要确切的匹配若干个单词或者短语(phrases)。例如我们想要查询同时包含"rock"和"climbing"（并且是相邻的）的员工记录。我们只要将 match 查询变更为 match_phrase 查询即可:
```json
{
  "query": {
    "match_phrase": {
      "about": "rock climbing"
    }
  }
}
```
毫无疑问，该查询返回John Smith的文档。

#### 8.高亮搜索结果
很多应用喜欢从每个搜索结果中高亮(highlight)匹配到的关键字，这样用户可以知道为什么这些文档和查询相匹配。
```json
{
  "query": {
    "match_phrase": {
      "about": "rock climbing"
    }
  },
  "highlight": {
    "fields": {
      "about": {}
    }
  }
}
```
当我们运行这个语句时，会命中与之前相同的结果，但是在返回结果中会有一个新的部分叫做 highlight ，这里包含了来自 about 字段中的文本，并且用 <em></em> 来标识匹配到的单词。

#### 9.聚合查询
最后，我们还有一个需求需要完成：允许管理者在职员目录中进行一些分析。 Elasticsearch有一个功能叫做聚合(aggregations)，它允许你在数据上生成复杂的分析统计。它很像SQL中的 GROUP BY 但是功能更强大。

在Elasticsearch5以后的版本中,聚合这些操作用单独的数据结构(fielddata)缓存到内存里了，需要单独开启, 执行分析查询之前要先执行一下操作
**PUT megacorp/_mapping/employee/**
```json
{
  "properties": {
    "interests": { 
      "type":     "text",
      "fielddata": true
    }
  }
}
```

然后进行聚合查询
**PUT megacorp/employee/_search**
```json
{
  "size": 0,
  "aggs": {
    "all_interests": {
      "terms": {
        "field": "interests"
      }
    }
  }
}
```

查询结果：
```json
{
    "aggregations":{
        "all_interests":{
            "doc_count_error_upper_bound":0,
            "sum_other_doc_count":0,
            "buckets":[
                {
                    "key":"music",
                    "doc_count":2
                },
                {
                    "key":"forestry",
                    "doc_count":1
                },
                {
                    "key":"sports",
                    "doc_count":1
                }
            ]
        }
    }
}
```
我们可以看到两个职员对音乐有兴趣，一个喜欢林学，一个喜欢运动。

当然聚合查询可以结合一般查，也可以使用分级汇总查询。

**对姓"Smith"的人的兴趣爱好统计**
```json
{
    "query":{
        "match":{
            "last_name":"smith"
        }
    },
    "aggs":{
        "all_interests":{
            "terms":{
                "field":"interests"
            }
        }
    }
}
```

**分级汇总:统计每种兴趣下职员的平均年龄**
```json
{
    "aggs":{
        "all_interests":{
            "terms":{
                "field":"interests"
            },
            "aggs":{
                "avg_age":{
                    "avg":{
                        "field":"age"
                    }
                }
            }
        }
    }
}
```

### 四、Elasticsearch集群
Elasticsearch的集群搭建非常简单。
只要第二个节点与第一个节点有相同的 cluster.name （请看 ./config/elasticsearch.yml 文件），它就能自动发现并加入第一个节点所在的集群。

学习的过程中没有多余的机器，也不想开多个虚拟机，只想在一台电脑上启动多个节点，这里需要额外增加一点配置。

**1.将Elasticsearch文件夹复制成3分**
并且修改每个文件夹的名字，比如elasticsearch-6.3.2， elasticsearch-6.3.2a, elasticsearch-6.3.2b

因为复制的elasticsearch文件夹下包含了data文件中先前示例的节点数据，需要把data文件下的文件清空。

**2.修改hosts文件**

添加一下IP映射
```hosts
127.0.0.1 peer1
127.0.0.1 peer2
127.0.0.1 peer3
```

**3.修改elasticsearch.yml**

依次打开三个elasticsearch中config目录下的下elasticsearch.yml配置文件，需要修改的位置如下：

```yaml
cluster.name: sj-cluster
node.name: node-1
network.host: peer1
http.port: 9201
transport.tcp.port: 9301
discovery.zen.ping.unicast.hosts: ["peer1:9301", "peer2:9302","peer3:9303"]
```

```yaml
cluster.name: sj-cluster
node.name: node-2
network.host: peer2
http.port: 9202
transport.tcp.port: 9302
discovery.zen.ping.unicast.hosts: ["peer1:9301", "peer2:9302","peer3:9303"]
```

```yaml
cluster.name: sj-cluster
node.name: node-3
network.host: peer3
http.port: 9203
transport.tcp.port: 9303
discovery.zen.ping.unicast.hosts: ["peer1:9301", "peer2:9302","peer3:9303"]
```

**4.依次启动3个Elasticsearch节点和插件**

集群部署完成， 打开http://localhost:9100/ 页面，会看到3个节点的信息。 前面学习的CRUD操作在集群里面是完全一样的。


