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
  * nscanned: 扫描的文档总数
  * millis: 耗时毫秒数
  * n: 结果数量
  * nscannedObjects: 查找实际文档的次数
  * nscanned: 查找过的索引条目数量
  * scanAndOrder: 是否进行了内存排序
  * indexOnly: 是否只使用索引就完成查询
  * nYields: 查询暂停的次数
  * indexBounds: 索引的使用情况
