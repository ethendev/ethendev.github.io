---
layout: post
title: Elasticsearch学习笔记之分片
tags:  [Elasticsearch]
categories: [Elasticsearch]
keywords: Elasticsearch,分片
---


为了将数据添加到Elasticsearch，我们需要`索引(index)`——一个存储关联数据的地方。实际上，索引只是一个用来指向一个或多个`分片(shards)`的"逻辑命名空间(logical namespace)".

## 分片
一个`分片(shard)`是一个最小级别“工作单元(worker unit)”,它只是保存了索引中所有数据的一部分。分片就是一个Lucene实例，并且它本身就是一个完整的搜索引擎。
我们的文档存储在分片中，并且在分片中被索引，但是我们的应用程序不会直接与它们通信，取而代之的是，直接与索引通信。


分片是Elasticsearch在集群中分发数据的关键。把分片想象成数据的容器。文档存储在分片中，然后分片分配到你集群中的节点上。当你的集群扩容或缩小，Elasticsearch将
会自动在你的节点间迁移分片，以使集群保持平衡。


分片可以是`主分片(primary shard)`或者是`复制分片(replica shard)`。你索引中的每个文档属于一个单独的主分片，所以主分片的数量决定了索引最多能存储多少数据。


> 理论上主分片能存储的数据大小是没有限制的，限制取决于你实际的使用情况。分片的最大容量完全取决于你的使用状况：硬件存储的大小、文档的大小和复杂度、如何索引和查询你的文档，以及你期望的响应时间。


复制分片只是主分片的一个副本，它可以防止硬件故障导致的数据丢失，同时可以提供读请求，比如搜索或者从别的shard取回文档。

当索引创建完成的时候，主分片的数量就固定了，但是复制分片的数量可以随时调整。每个索引的主分片数，默认值是 5。



## 分片内部原理编辑
在 集群内的原理, 我们介绍了 分片, 并将它 描述成最小的 工作单元_。但是究竟什么 _是 一个分片，它是如何工作的？ 在这个章节，我们回答以下问题:

为什么搜索是 近 实时的？
为什么文档的 CRUD (创建-读取-更新-删除) 操作是 实时 的?
Elasticsearch 是怎样保证更新被持久化在断电时也不丢失数据?
为什么删除文档不会立刻释放空间？
refresh, flush, 和 optimize API 都做了什么, 你什么情况下应该是用他们？
最简单的理解一个分片如何工作的方式是上一堂历史课。 我们将要审视提供一个带近实时搜索和分析的 分布式持久化数据存储需要解决的问题。

