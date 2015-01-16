---
layout: post
title: MongoDB中计算聚合值的三种方法（一）
date: 2012-12-31 22:25 +08:00
tags: Tech, MongoMapper, MongoDB
---

说到报表，不可避免的会涉及到计算数字域的总计、平均值、最大值和最小值。前一阵在做<a href="https://jinshuju.net" target="_blank">金数据</a>的过程中，对使用MongoMapper做聚合做了一点点研究。总的来说，应该是有三种方法：

<h4>1. 使用Map-Reduce</h4>

MongoDB的Map-Reduce可谓是功能强大，可以处理复杂的聚合逻辑。关键是map方法和reduce方法的设计。

{% highlight ruby %}
def map field
  <<-MAP
    function() {
      var number = this.#{field};
      emit(1, {min: number, max: number, sum: number, count: 1});
    }
  MAP
end
{% endhighlight %}

这里，我们不需要对任何域做group，所以emit的第一个参数就给了1，而第二个参数则构造了一个包含最大值、最小值、和值和计数的对象，这个对象的结构必须跟下面的reduce方法返回的对象结构一致：

{% highlight ruby %}
def reduce
  <<-REDUCE
    function(key, values) {
      var firstValue = values[0];
      var result = {min: firstValue.min, max: firstValue.max, sum: 0, count: 0};
      values.forEach( function (value) {
        if (value.min < result.min) result.min = value.min;
        if (value.max > result.max) result.max = value.max;
        result.count += value.count;
        result.sum += value.sum;
      });
      return result;
    }
  REDUCE
end
{% endhighlight %}

最后，我们就可以利用finalize方法根据和值和计数求得平均值：

{% highlight ruby %}
def finalize
  <<-FINALIZE
    function(key, result) {
      result.avg = result.sum / result.count;
      delete result.count;
      return result;
    }
  FINALIZE
end
{% endhighlight %}

上面不论是reduce方法还是finalize方法，定义的js function中的第一个参数key，都是与map方法中emit的第一个参数对应的，也就是我们需要按其进行分组的域。
例如我们要对数据的日期进行分组，那么在map方法中，emit的第一个参数就应该为this.date_field。

如此，就可以调用MongoMapper的API了：

{% highlight ruby %}
def number_statistics field, condition
  self.collection.map_reduce(
    map(field), reduce,
    :out => "number_statistics_result",
    :query => condition,
    :finalize => finalize
  ).find.first
end
{% endhighlight %}