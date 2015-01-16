---
layout: post
title: Cross Domain iFrame Auto Height
date: 2013-04-30 16:00 +08:00
tags: Tech, UI
---

We have a feature, that is providing a js script which generates an iframe containing a form, so that other people can embed this form on their website by just including this js.
What we did before, is calculating the form height, and writing the height into the script, then the host website could know how much the iframe's height should be.
However sometimes people will change the form, so the height will be changed! Then they need to update the script on their website, otherwise there will be an ugly scroll bar when the height increases.
This is really rebarbative 'cause we can't force our user to update the script every time they change the form.

A few days ago, we assigned this issue with a high priority, which was making the iframe set it's height automatically by it's own content height.

So the hardest problem is how can we deal with the cross domain security issue, since we can't get the iframe content height from the parent page if they are in different domain.
Well luckily, after a few digging, we came up with the **postMessage** way! Can't visit the content window of a different domain? fine, then we can send a message from the iframe!

Here is the code:

in the iframe, add the following script(in coffee)
{% highlight coffee %}
$ ->
  postSize = ->
    target = if parent.postMessage
      parent
    else
      if parent.document.postMessage then parent.document else undefined
    target.postMessage $(document).height(), "*" if target
  if window.addEventListener
    window.addEventListener "load", postSize, false
  else
    window.attachEvent "onload", postSize, false if window.attachEvent
{% endhighlight %}

Let me explain the above code. There is a *postSize* function, which finds the parent, then posts the height as a message to the parent.
This postSize function will be bound as soon as the window loaded. *window.attachEvent* is used to deal with the compatibility of IE.

Then in the parent site, add the following script before the iframe
{% highlight javascript %}
receiveSize = function (e) {
  if ('< your iframe domain >'.indexOf(e.origin) === 0) {
  // to make sure it's the message posted from the right iframe
    document.getElementById(iframeId).style.height = e.data + 'px';
  }
};
if (window.addEventListener) {
  window.addEventListener('message', receiveSize, false);
} else if (window.attachEvent) {
  window.attachEvent('onmessage', receiveSize, false);
}
{% endhighlight %}

Here we use the pure javascript, similar, the parent site listens to the message event, then sets the iframe height.

Voila, we achieved it ~ Then we no longer suffered the pain of iframe height ~