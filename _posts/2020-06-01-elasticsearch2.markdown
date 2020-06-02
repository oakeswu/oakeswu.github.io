---
layout:     post
title:      "ES生产优化经验"
subtitle:   ""
date:       2020-06-01
author:     "OakesWu"
header-img: "img/post-bg-2015.jpg"
tags:
    - ElasticSearch
---

# 背景
因为业务需求需要在生产中多次使用到ES，在长期运用中也经历到一些问题并解决，所以希望通过文章记录下经验并分享

# 写数据优化
- 使用bulk批量写索引方式减少http请求次数
- 禁止动态mapping，只对有需要的字段做索引映射，明确好每个字段的类型
- 避免使用nested类型索引，因为nested索引数据是单独存放的一个文件，当nested文件很大的时候会拖累整个查询性能
- 数字类型和keyword类型要依情况判断，在ES5.x之后的**数字类型**的索引结构不再是倒排索引，而是Block k-d tree的索引结构，此时数值类型的索引做范围查找的性能较高，Term查询的性能就会变慢，具体原理可以参考公司大佬kennywu的一篇文章（[https://elasticsearch.cn/article/446](https://elasticsearch.cn/article/446)）

# 搜索优化
- 使用Term查询的性能最佳
- 避免使用Script查询和Script排序，当召回量很大的时候性能会极差，因为脚本一般都需要经过实时计算。
- 善用filter查询，因为filter过滤查询是有缓存的

# 数据一致性
ES运行一般是分布式集群模式，那么就会涉及到CAP理论，ES使用的AP，当索引分片在做复制的时候是无法保证数据一致性的，所以如果业务需要强一致性需求，可能我们需要考虑业务上解决而不是ES。