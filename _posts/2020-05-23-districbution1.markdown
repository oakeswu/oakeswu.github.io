---
layout:     post
title:      "分布式(一) -- CAP理论"
subtitle:   ""
date:       2020-05-23
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - 分布式
---

# 什么是CAP
之前有同事问我ES在分片复制的时候如何保证数据一致性，因为ES可理解成高可用的分布式集群组成的搜索服务，既然分布式就绕不开数据一致性那个的问题，其实ES集群并没有保证数据一致性，只能在业务代码中进行处理。此时就会涉及到分布式一个经典理论-CAP。CAP理论是分布式系统的基本定理，分别对应着三个单词Consistency（一致性），Availability（可用性），Partition tolerance（分区容错性）。特别注意的是这三个特性最多只能满足两个，不可能同时满足。
# 为什么不能同时满足
参考（[michael whittaker文章](https://mwhittaker.github.io/blog/an_illustrated_proof_of_the_cap_theorem/)）
首先我们有一个集群，里面包含G1服务器和G2服务器，G1和G2里面参数值相同，都为V0.
![cluster](/img/doc/distribution/distribution1one.png)
- P（分区容错性）
分区容错性是指当集群中任意一个分区到另一个分区发送消息都有可能因为网络问题而丢失，上面的分区可以理解成服务器之前，一般来说分区容错性的问题无法避免，所以涉及分布式系统的时候都会考虑满足P
- C（一致性）
一致性是指当客户端读取服务器的值必须是服务器最新的值即客户端更新了服务器的值，当客户端读取服务器时必须返回刚更新后的值。
![write](/img/doc/distribution/distribution1two.png)
![read](/img/doc/distribution/distribution1three.png)
此时由于集群有两台服务器G1和G2，**当G1一开始更新值并发消息给G2同步数据，但是由于分区容错性里面的网络原因延时更新或失败**，那么client读取集群数据时，就有可能返回G1的值V1和G2的值V0。此时如果保证一致性就需要**暂停G2的读写操作并等待G1的数据同步到G2完成**
![synchronous](/img/doc/distribution/distribution1four.png)
![synchronous](/img/doc/distribution/distribution1five.png)
- A（可用性）
可用性是指集群非故障节点服务器收到的每个请求都必须响应，即不能出现一致性里面因数据同步暂停G2服务的情况。所以此时我们可以发现CA理论在存在P的情况下是无法同时满足的。所以现在很多分布式系统为了高可用演进成了保证数据最终一致性。