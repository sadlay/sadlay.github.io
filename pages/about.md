---
layout: page
title: About
description: Focus on Coding Bug >_<
keywords: Sadlay, Lay
comments: true
menu: 关于
permalink: /about/
---

我是Lay，专注写Bug。

Never Give Up!


## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
