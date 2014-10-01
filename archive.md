---
layout: page
title: Archives
permalink: /archive/
---
{% assign post_year1 = "" %}
{% for post in site.posts %}{% capture post_year2 %}{{ post.date | date: '%Y' }}{% endcapture %}{% if post_year1 != post_year2 %}{% assign post_year1 = post_year2 %}

##{{ post_year1 }}

{% endif %}
<span class="pull-right">{{ post.date | date: "%F" }}</span>[{{ post.title }}]({{ post.url }})

{% endfor %}