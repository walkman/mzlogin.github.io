---
layout: post
title: SpringCloud Gateway 智能网关
categories: [SpringCloud,Gateway]
excerpt: Spring Cloud Gateway是Spring Cloud官方推出的第二代网关框架，取代Zuul网关。网关作为流量的，在微服务系统中有着非常作用，网关常见的功能有路由转发、权限校验、限流控制等作用。
tags: Gateway
---

### 一、简介
Spring Cloud Gateway是Spring Cloud官方推出的第二代网关框架，取代Zuul网关。

网关作为一个系统的流量的入口，有着举足轻重的作用，通常的作用如下：
* 协议转换，路由转发
* 流量聚合，对流量进行监控，日志输出
* 作为整个系统的前端工程，对流量进行控制，有限流的作用
* 作为系统的前端边界，外部流量只能通过网关才能访问系统
* 可以在网关层做权限的判断
* 可以在网关层做缓存

Spring Cloud Gateway作为Spring Cloud框架的第二代网关，在功能上要比Zuul更加的强大，性能也更好。
随着Spring Cloud的版本迭代，Spring Cloud官方有打算弃用Zuul的意思。
Spring Cloud Gateway的使用和功能上，Spring Cloud Gateway替换掉Zuul的成本上是非常低的，几乎可以无缝切换。
Spring Cloud Gateway几乎包含了zuul的所有功能。

![Gateway架构图](/images/posts/frameworks/spring_cloud_gateway_diagram.png)

如上图所示，客户端向Spring Cloud Gateway发出请求。 如果Gateway Handler Mapping确定请求与路由匹配（这个时候就用到predicate），
则将其发送到Gateway web handler处理。 Gateway web handler处理请求时会经过一系列的过滤器链。 
过滤器链被虚线划分的原因是过滤器链可以在发送代理请求之前或之后执行过滤逻辑。 先执行所有“pre”过滤器逻辑，然后进行代理请求。 
在发出代理请求之后，收到代理服务的响应之后执行“post”过滤器逻辑。这跟zuul的处理过程很类似。在执行所有“pre”过滤器逻辑时，
往往进行了鉴权、限流、日志输出等功能，以及请求头的更改、协议的转换；转发之后收到响应之后，会执行所有“post”过滤器的逻辑，
在这里可以响应数据进行了修改，比如响应头、协议的转换等。


