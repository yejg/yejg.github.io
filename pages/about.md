---
layout: page
title: About
description: 钢仔
keywords: yejg, yejg1212, 钢仔
comments: true
menu: 关于
permalink: /about/
---

我是钢仔，程序猿一枚

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

[^_^]:
## Skill Keywords
{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
