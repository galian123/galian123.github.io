---
layout: page
title: Tags
published: false
---

{% comment %}
From:
 http://codinfox.github.io/dev/2015/03/06/use-tags-and-categories-in-your-jekyll-based-github-pages/
{% endcomment %}

{% assign rawtags = "" %}
{% for post in site.posts %}
	{% assign ttags = post.tags | join:'|' | append:'|' %}
	{% assign rawtags = rawtags | append:ttags %}
{% endfor %}
{% assign rawtags = rawtags | split:'|' | sort %}

{% assign tags = "" %}
{% for tag in rawtags %}
	{% if tag != "" %}
		{% if tags == "" %}
			{% assign tags = tag | split:'|' %}
		{% endif %}
		{% unless tags contains tag %}
			{% assign tags = tags | join:'|' | append:'|' | append:tag | split:'|' %}
		{% endunless %}
	{% endif %}
{% endfor %}

通过Tag来检索文章。

{% for tag in tags %}
[{{tag}}]({{site.url}}/tags/#{{ tag | slugify }}) {% endfor %}

{% for tag in tags %}
<div id="{{ tag | slugify }}"><center>{{ tag }}</center></div>

<ul>
{% for post in site.posts %}
{% if post.tags contains tag %} <li> {{post.date | date: "%Y-%m-%d"}} <a href="{{ post.url }}">{{ post.title }}</a> </li> {% endif %}
{% endfor %}
</ul>

{% endfor %}
