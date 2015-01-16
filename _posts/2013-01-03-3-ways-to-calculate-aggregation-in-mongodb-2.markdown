---
layout: post
title: MongoDB中计算聚合值的三种方法（二）
date: 2013-01-03 13:36 +08:00
tags: Tech, MongoMapper, MongoDB
---

在[上一篇博文]({% post_url 2012-12-31-3-ways-to-calculate-aggregation-in-mongodb-1 %})中，介绍了如何使用Map-Reduce计算聚合值。
但我们的需求并不需要进行任何的映射，我们只要对过滤出来的结果进行计算就可以了。这样，MongoDB中的group方法就可以直接为我所用。

<h4>2. 使用group</h4>

逻辑上跟使用Map-Reduce差不多，不过reduce和finalize的参数有一些变化。
{% highlight ruby %}
def number_statistics field, condition
  self.collection.group(
      :cond => condition,
      :reduce => number_reduce(field),
      :initial => {sum: 0, count: 0},
      :finalize => finalize
  ).find.first
end
{% endhighlight %}

上面是group方法的参数结构。cond与Map-Reduce中的query参数一样，是查询条件；增加了initial，其值为结果初始对象。我们这里将和值和计数置为0。

reduce方法的参数则为当前文档(current)和结果对象(result)；而在Map-Reduce中，第一个参数为key，第二个参数为map方法中映射到该key的所有对象。

{% highlight ruby %}
def number_reduce field
  <<-REDUCE
    function(current, result) {
      var value = current.#{field};
      if (result.min == null || result.min > value) result.min = value;
      if (result.max == null || result.max < value) result.max = value;
      result.sum += value;
      result.count++;
    }
  REDUCE
end
{% endhighlight %}

finalize方法也有所不同，不需要返回值，传递的参数即为结果对象，我们只需要对其进行修改就可以了：

{% highlight ruby %}
def finalize
  <<-FINALIZE
    function(result) {
      result.avg = result.sum / result.count;
      delete result.count;
    }
  FINALIZE
end
{% endhighlight %}

相比较上一种方法，此方法已简单许多，但是对于复杂的聚合逻辑，它也许就不如Map-Reduce能驾驭了。