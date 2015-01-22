---
layout: post
title: 一些MongoDB索引的设计规则
date: 2015-01-22 22:35 +08:00
tags: Tech, MongoDB, Index
published: false
---

在讲到规则之前，先介绍一下怎么查看索引的使用情况（以下代码示例均为ruby + mongoid）：

1. 对于一般查询，在查询语句后加explain即可，比如`form.entries.desc(:created_at).explain`。返回的需要我们重点关注的各项指标如下：
  * cursor: 执行过程中的命中索引
  * millis: 耗时毫秒数
  * n: 结果数量
  * nscannedObjects: 命中索引下查找实际文档的次数
  * nscanned: 命中索引下查找过的索引条目数量
  * scanAndOrder: 是否进行了内存排序
  * indexOnly: 是否只使用索引就完成查询
  * nYields: 查询暂停的次数
  * indexBounds: 索引的使用情况
  * allPlans: 所有测试过的索引。如果mongodb发现有多个索引都能够匹配到当前查询条件，那么会同时向这多个索引发送查询请求，最先返回100个文档的索引将会被命中。
 
 那么，这些指标怎么看呢？首先，我们肯定希望millis越小越好，scanAndOrder尽量为false，nscanned和nscannedObjects同样是越接近n越好。如果索引能够覆盖整个查询条件，那么indexOnly也会为true从而不用再去扫描实际文档。如果设计的够好，allPlans也只包含被命中的索引，从而减少额外的数据库开销。

