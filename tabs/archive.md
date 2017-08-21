---
layout: page
title: "文章存档"
description: ""
header-img: "img/orange.jpg"
---

<ul class="listing">
{% for post in site.posts %}
  {% capture y %}{{post.date | date:"%Y"}}{% endcapture %}
  {% if year != y %}
    {% assign year = y %}
    <h2 class="listing-seperator">{{ y }}</h2>
  {% endif %}
  <li class="listing-item item">
    <time datetime="{{ post.date | date:"%Y/%m/%d" }}">{{ post.date | date:"%Y/%m/%d" }}</time>    
    <a href="{{ post.url }}" title="{{ post.title }}">{{ post.title }}</a>
  </li>
{% endfor %}
</ul>

<style>
.item {
  margin : 5px;
  color : #666;
}
</style>
