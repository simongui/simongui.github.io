---
layout: default
title: Simon Guindon
---

# Articles
[Information on distributed systems](distributed-systems)

# Posts
<ul id="posts" class="twelve columns offset-by-four">
  {% for post in site.posts %}
    {% if post.tags contains 'draft' %}
      <!-- This is a draft. Allow blog to be displayed in draft blog lists. Use in blog page layout to display a warning that the page is a draft. -->
    {% else %}
      <!-- This is not a draft. Allow post to be displayed in blog lists and RSS feed -->
      <li>
        {% if post.external_url %}
          <a class="nine columns" href="{{ post.external_url }}">{{ post.title }}</a>
        {% else %}
          <a class="nine columns" href="{{ post.url }}">{{ post.title }}</a>
        {% endif %}

        <span class="two columns">{{ post.date | date: "%B %d, %Y" }}</span>
      </li>
    {% endif %}
  {% endfor %}
</ul>
