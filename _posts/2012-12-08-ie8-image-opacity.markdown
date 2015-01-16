---
layout: post
title: IE8下的图片透明度
date: 2012-12-08
tags: Tech, UI, IE
---

有这样一个需求：有一个数据表格，要求hover到一条数据上时显示可以对该条数据进行的操作，譬如删除和查看，用一个个小图标来表示。而hover到单个图标上时，图标颜色加深。如下图：

![hover icon](/images/2012-12-08-ie8-image-opacity/hover-icon.png)

一开始，我们的解决方式是：一个深色的png图标，默认给一个0.6的opacity（IE下是alpha），hover上去的时候将opacity置回1。

{% highlight scss %}
a {
  .icon {
    background: url("icon.png");
    @include square(16px);
    @include opacity(0.6);
  }

  &:hover {
    .icon {
      @include opacity(1);
    }
  }
}
{% endhighlight %}

这种方式在chrome等其他浏览器上都表现完美，但在IE8上简直差劲极了，所有带弧度的地方，由于有半透明的灰度，都被渲染成了黑色：

![hover icon ie8](/images/2012-12-08-ie8-image-opacity/hover-icon-ie8.png)

早在IE5、6的时代，PNG图片的渲染确实是有问题的，为此，还专门有人写了一个pngfix的js库。我以为，这是同样的问题啊，于是尝试应用该库去解决，未果。
又google了一下png图片在IE8的css fix方式，用神奇的ImageAlphaLoader也同样未果。
甚至又开始怀疑这个png图片素材制作的精度有问题，因为其他的素材都能够以这种方式在IE8下很好的渲染出来。

真的是上述原因吗？为了便于调试，我把opacity去掉了，竟然惊讶的发现，圆角处的半透明都能完美地渲染出来，突然顿悟了：IE8下，png半透明 + alpha是不行的，如果png图片只是纯色，ok没问题。
好吧，最后的解决方案：两个版本的素材，一个浅一些，然后借助CSS sprite来完成。

所以最终的解决方案是：

{% highlight scss %}
a {
  .icon.trash {
    background: url("icon.png") 0px -326px;
    @include square(16px);

    &:hover {
      background-position: 0px -342px;
    }
  }
}
{% endhighlight %}