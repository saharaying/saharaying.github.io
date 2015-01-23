---
layout: post
title: MongoDB索引设计的一些规则
date: 2015-01-22 22:35 +08:00
tags: Tech, MongoDB, Index
---

在讲到规则之前，先介绍一下怎么查看索引的使用情况（以下代码示例均为ruby + mongoid）：

1. 对于一般查询，在查询语句后加explain即可，比如`form.entries.desc(:created_at).explain`。返回的需要我们重点关注的各项指标如下：
  * cursor: 执行过程中的命中索引
  * millis: 耗时毫秒数
  * n: 结果数量
  * nscannedObjects: 按照索引指针在磁盘上查找实际文档的次数
  * nscanned: 查找过的索引条目数量
  * scanAndOrder: 是否进行了内存排序
  * indexOnly: 是否只使用索引就完成查询
  * nYields: 查询暂停的次数
  * indexBounds: 索引的使用情况
  * allPlans: 所有测试过的索引。如果mongodb发现有多个索引都能够匹配到当前查询条件，那么会同时向这多个索引发送查询请求，最先返回100个文档的索引将会被命中。
 
    那么，这些指标怎么看呢？首先，我们肯定希望millis越小越好，scanAndOrder尽量为false，nscanned和nscannedObjects同样是越接近n越好。如果索引能够覆盖整个查询条件，那么indexOnly也会为true从而不用再去扫描实际文档。如果设计的够好，allPlans也只包含被命中的索引，从而减少额外的数据库开销。

2. 如果要查看聚合操作（aggregation）的索引：
`collection_class.collection.session.command(aggregate: collection_class.collection.name, pipeline: pipeline.flatten, explain: true)`

3. 另外，可以实时监控当前数据库的状态：`mongotop` 和 `mongostat`

现在讲一些有可能会被忽视的索引设计规则和查询语句的写法：

1. 注意索引基数。索引基数是指集合中某个字段拥有不同值的数量。一个字段的基数越高，这个键上的索引就越有用。所以应该在基数比较高的键上建立索引，或者至少应该把基数较高的键放在复合索引的前面。比如不应该对一个boolean值的字段建立索引。

2. 对于复合索引，应该将会用于精确匹配的字段放在索引的前面，将用于范围匹配的字段放在最后。比如，查询一个表单1小时前提交的数据 `form.entries.lt(created_at: 1.hour.ago)`，针对这种查询，应当建立索引 `Entry.index form_id: 1, deleted_at: 1, created_at: 1`。这里会有deleted_at是因为我们使用了Mongoid::Paranoia来实现软删除，会建立default_scope`{deleted_at: nil}`。

3. `$or`操作可以对每个子句都使用索引，它实际上是执行多次查询然后将结果集合并。所以查询语句中应该尽可能使用`$in`而不是`$or`，比如使用`Entry.in(form_id: [form1.id, form2.id])`而不是`Entry.or({form_id: form1.id}, {form_id: form2.id})`。

4. 对于count操作，[官方文档](http://docs.mongodb.org/manual/reference/method/db.collection.count/#index-use)上是这样说的：
    
    When performing a count, MongoDB can return the count using only the index if:

    * the query can use an index,
    * the query only contains conditions on the keys of the index, and
    * the query predicates access a single contiguous range of index keys.


    所以假如我们在entries上有`{form_id: 1, deleted_at: 1, created_at: 1}`的索引，计算`form.entries.lt(created_at: 1.hour.ago).count`，是能仅根据命中索引就能返回值的，而如果是`form.entries.lt(serial_number: 1000).count`则不能仅使用索引得到，因此性能上就会差一些。

