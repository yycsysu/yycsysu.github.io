---
layout: post
title: "A Post with a Video"
description: "Custom written post descriptions are the way to go... if you're not lazy."
tags: [sample post, video]
---

<iframe style="width:704px;height:436px;" src="https://ssl.acfun.tv/block-player-homura.html#vid=2472444;from=http://www.acfun.tv" id="ACFlashPlayer-re" frameborder="0"></iframe>
Video embeds are responsive and scale with the width of the main content block with the help of [FitVids](http://fitvidsjs.com/).

Not sure if this only effects Kramdown or if it's an issue with Markdown in general. But adding YouTube video embeds causes errors when building your Jekyll site. To fix add a space between the `<iframe>` tags and remove `allowfullscreen`. Example below:

{% highlight html %}
<iframe style="width:704px;height:436px;" src="https://ssl.acfun.tv/block-player-homura.html#vid=2472444;from=http://www.acfun.tv" id="ACFlashPlayer-re" frameborder="0"></iframe>
{% endhighlight %}