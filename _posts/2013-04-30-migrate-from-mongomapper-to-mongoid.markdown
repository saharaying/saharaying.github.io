---
layout: post
title: MongoMapper迁移到Mongoid
date: 2013-04-30 15:47 +08:00
tags: Tech, MongoMapper, Mongoid, MongoDB
---

这篇博客断断续续写了将近一个月，终于算是总结的差不多了，将就着看吧。

--------

原本我们使用了MongoMapper作为MongoDB的ORM，但是随着业务逻辑越来越复杂，它已经不能满足我们的需求了:

1. 数据量越来越大，数据库查询性能问题越来越显著，据说Mongoid的性能可以提高4倍
2. MongoMapper的gem最后一次更新是在2012年9月，维护的频率不如Mongoid（最后更新2013年4月17日），并且对动态域的存取操作支持也不好
3. 围绕Mongoid的周边gem更加丰富

于是我花了大概4天的时间将项目从MongoMapper迁移到了Mongoid。能看得到的明显的变化有:

* 数据库查询确实快了很多，包括聚合查询的速度也有很大提升。
* 调用*update_attributes*方法，不再是使用MongoDB的*$update*更新所有域，而是使用*$set*仅更新有变化的域。

在迁移过程中，需要注意以下几个方面：

#### 1. 数据库连接配置
在mongid.yml中，我们配置了两个option： *include_type_for_serialization: true* 和*raise_not_found_error: false*。

第一个是指在对model进行序列化时（比如to_json），默认包含Single Collection Inheritance的_type字段。

第二个顾名思义，当使用find方法查询model时，如果找不到，返回nil，而不是抛出异常。
{% highlight yaml %}
development:
  sessions:
    default:
      hosts:
        - 127.0.0.1:27017
      database: goldendata_development
      username: root
      password:
  options:
    include_type_for_serialization: true
    raise_not_found_error: false
{% endhighlight %}

#### 2. Model中的字段和relation
- field和fields是关键字，不能作为属性或relation的名字，如果数据库里对应的key为*fields*，那么你需要**store_as**这个option。譬如
*embeds_many :f_fields, store_as: 'fields', class_name: 'Field'*
- 如果要给model加上时间戳，只需要**include Mongoid::Timestamps**这个module就可以了
- 没有userstamps的module，如果真的需要，只好自己实现
- Mongoid**支持动态域的存取**，也就是说，不用*field*关键字声明的属性也可以被访问和保存，只需要调用*read_attribute*或*write_attribute*方法即可，譬如 *model.read_attribute(:dynamic_field)* 或者 *model\[:dynamic_field\]*，如果动态域不存在，则会返回nil。

#### 3. 查询和聚合
- 复杂的查询由**[origin](http://mongoid.org/en/origin/docs/selection.html)**提供，包括and/or/elem_match/gt等一系列方法。譬如
*Entry.or({field1: 'a'}, {field2: 'b'})*
- sort需要替换成对应的**asc**/**desc**或**order_by**，比如*desc(:created_at)* 或者 *order_by(:created_at.desc)*
- 另外，Mongoid不再支持group方法，应该改用aggregate或者map_reduce实现

#### 4. 其他model相关的变更
- 异常。*MongoMapper::DocumentNotFound* 异常要换成 *Mongoid::Errors::DocumentNotFound*
- *to_mongo*和*from_mongo*方法要分别换成**mongoize**和**demongoize**
- 对model进行序列化时，默认的与id对于的key不再是*id*，而是<em>_id</em>
- 对于callback，譬如after_create，如果这个model上含有has_many的relation，并且该relation设置为autosave: true，那么这个after_create的callback将会发生在该model创建以后，但是在这个has_many的关联保存之前。参见：
> Note that to be efficient, Mongoid only fires the callback of the document that the persistence action was executed on. This is that Mongoid aims to support large hierarchies and to handle optimized atomic updates callbacks can't be firing all over the document hierarchy.
- Mongoid对model有着更严格的检查，比如我们不能直接创建一个embedded document对象
- 创建**索引**的方式改变。在Mongid中，调用*Form.index(creator_id: 1, created_at: -1)*不会真正创建索引，必须运行*rake db:mongoid:create_indexes*，而要移除索引，运行*rake db:mongoid:remove_indexes*。

#### 5. ORM周边的gem替换
如果使用了一些需要依赖MongoDB的gem，比如omniauth-identity, will_paginate, mm-carrierwave, sidekiq等，

- *include OmniAuth::Identity::Models::MongoMapper* 变为 *include OmniAuth::Identity::Models::Mongoid*
- will_paginate gem换成will_paginate_mongoid
- mm-carrierwave gem换成carrierwave-mongoid，但是如果只是声明成*mount_uploader &lt;mounted_as&gt;, Uploader*，文件名将会存储到&lt;mounted_as&gt;字段，而不是像mm-carrierwave那样存储到&lt;mounted_as&gt;\_filename。为了不进行数据迁移，我们可以加上**mount_on**的声明：
*mount_uploader &lt;mounted_as&gt;, Uploader, mount_on: &lt;mounted_as&gt;\_filename*。
- 如果使用了sidekiq，必须确保每个sidekiq的worker运行完后释放数据库的连接，那么可能需要使用**[kiqstand](http://mongoid.org/en/mongoid/docs/tips.html#sidekiq)**。