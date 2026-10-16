---
layout: page
title: Blog
subtitle: 
---

<div>
{% assign tags = site.tags | sort %}
<div class="tag-filter">
<a class="tag" href="{{ '/blog' | prepend: site.baseurl }}" data-tag="">all</a>
{% for tag in tags %}
<a class="tag" href="{{ '/blog' | prepend: site.baseurl }}?tag={{ tag[0] | uri_escape }}" data-tag="{{ tag[0] | escape }}">{{ tag[0] }}</a>
{% endfor %}
</div>
<ul class="list-posts">
{% for post in site.posts %}
<li class="post-teaser" data-tags="{{ post.tags | join: '|' | escape }}">
<a href="{{ post.url | prepend: site.baseurl }}">
<span class="post-teaser__title">{{ post.title }}</span>
<span class="post-teaser__date">{{ post.date | date: "%d %B %Y" }}</span>
</a>
{% include tags.html tags=post.tags %}
</li>
{% endfor %}
</ul>
</div>

<script>
(function () {
  var tag = new URLSearchParams(window.location.search).get('tag') || '';
  document.querySelectorAll('.tag-filter .tag').forEach(function (el) {
    el.classList.toggle('tag--active', el.dataset.tag === tag);
  });
  if (!tag) return;
  document.querySelectorAll('.post-teaser').forEach(function (el) {
    el.hidden = el.dataset.tags.split('|').indexOf(tag) === -1;
  });
})();
</script>
