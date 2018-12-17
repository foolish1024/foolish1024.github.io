---
layout: page
title: About
description: 一个技术小白的自留地
keywords: foolish024
comments: true
menu: 关于
permalink: /about/
---

不断努力，学习的东西会不断更新记录在这里。

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
