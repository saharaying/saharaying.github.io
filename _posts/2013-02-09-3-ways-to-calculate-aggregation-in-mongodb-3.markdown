---
layout: post
title: MongoDB中计算聚合值的三种方法（三）
date: 2013-02-09 20:35 +08:00
tags: Tech, MongoMapper, MongoDB
---

其实，MongoDB已经提供了求最大最小平均值和和值的方法，所以连reduce都可以免了，我们可以直接使用aggregation框架，这就是第三种方法。

<h4>3. 使用aggregation框架</h4>

MongoDB文档中[aggregation这一节](http://docs.mongodb.org/manual/reference/aggregation)对这个框架的使用有详细的说明。结合我们的例子，可以这样使用：

{% highlight ruby %}
def number_statistics field, condition
  self.collection.aggregate([
    {:$match => condition.merge(field => {:$ne => nil})},
    {:$group => {
      :_id => "$form_id",
      :min => {:$min => "$#{field}"},
      :max => {:$max => "$#{field}"},
      :sum => {:$sum => "$#{field}"},
      :avg => {:$avg => "$#{field}"}
    }},
    {:$project => {:_id => 0, :min => 1, :max => 1, :sum => 1, :avg => 1}}
]).first
end
{% endhighlight %}

下面对这段代码做以说明，用到了3个操作符：$match，$group和$project。
<ol>
<li>
    <h5>$match</h5>
    对需要进行聚合运算的文档进行过滤，MongoDB中接受一个javascript object作为值，如果使用MongoMapper，那么可以是一个hash。
    我们这里需要过滤掉空值，也就是说不希望空值参与聚合计算，所以有 <code>field => {:$ne => nil}</code>。
</li>
<li>
    <h5>$group</h5>
    我们需要的运算都在这里了，_id是必须的，表明要根据什么进行group，接下来就是要计算并返回的min、max、sum和avg。
</li>
<li>
    <h5>$project</h5>
    这里对返回结果进行了打磨，<code>:_id => 0</code> 表明我们在最终结果里去掉了_id，而其他计算值则会返回。
</li>
</ol>

需要注意一点的是，$match是pipeline中的第一个操作，这样就可以在做聚合之前先过滤掉我们不需要的文档，提高处理的速度。

Voilà, 就到这里吧，大过年的，休息，休息，休息一下～