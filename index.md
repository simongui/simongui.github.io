---
layout: default
title: Simon Guindon
---

# Articles
[Information on distributed systems](distributed-systems)

# Posts
<ul id="posts" class="twelve columns offset-by-four">
  {% for post in site.posts %}
    {% unless post.draft %}
      <li>
        {% if post.external_url %}
          <a class="nine columns" href="{{ post.external_url }}">{{ post.title }}</a>
        {% else %}
          <a class="nine columns" href="{{ post.url }}">{{ post.title }}</a>
        {% endif %}
        <span class="two columns">{{ post.date | date: "%B %d, %Y" }}</span>
      </li>
    {% endunless %}
  {% endfor %}
</ul>
