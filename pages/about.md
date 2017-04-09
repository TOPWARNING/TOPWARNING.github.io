---
layout: page
title: About
description: 码动人生
keywords: Warning
comments: true
menu: 关于
permalink: /about/
---

我是Warning，Android撸sir。

需要懂奋斗，更需要懂做人。

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
