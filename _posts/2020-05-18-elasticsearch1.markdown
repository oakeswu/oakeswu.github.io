---
layout:     post
title:      "ElasticSearch简介"
subtitle:   ""
date:       2020-05-18
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - ElasticSearch
---

# 什么是ES
ElasticSearch简称ES，是基于Lucene开发的一套分布式的查询和分析引擎，简便了开发者创建搜索系统的难度，可有效的存储，搜索并分析数据。
Lucene是目前最先进，最高效的开源搜索引擎框架。ES通过简单的RESTful api隐藏了Lucene的复杂性，让搜索开发变得简单，可以提供近实时搜索，近实时分析，并可分布式部署和扩展，拥有良好的故障处理机制。

#基本概念
- 集群（cluster）
集群是一组具有相同cluster.name配置的节点集合
- 节点（node）
节点是运行着ElasticSearch的实例，一般为一台机器
- 索引（index）
索引是一系列document的集合
- 文档（document）
文档是一个可被索引的基础信息单元，以json格式表示
- 分片（shards）
文档分布在各个分片上存储，分片又被放到各个节点机器，每个分片可以进行增删改查，便于横向扩展和并行操作
- 复制分片（replicas）
可以理解成备份分片，主要防止分片故障，加速查询索引，提高可用性

# 基本原理
-写文档
![write document](/img/doc/es/es1one.png)

1.根据文档ID和routing计算出该存放的分片上并转发请求到该分片
2.该分片验证请求并将转发请求到其他节点的复制分片上同步数据
3.当复制分片请求完成之后返回响应到主分片，然后主分片封装成功的响应给client

- 读文档

![image.png](/img/doc/es/es1two.png)

1：client查询请求发送到任意节点，接收到的节点会叫协调节点（coordinating node）
2：协调节点解析查询请求，向其他关联分片转发请求，一个主分片和它的复制分片任意选择
3：各数据分片检索本地数据并返回协调节点，经汇聚处理后返回请求client

# 总结
ES利用了倒排索引提升查询效率，在公司中使用率很高，在使用过程中也发现了许多可以优化查询的地方。