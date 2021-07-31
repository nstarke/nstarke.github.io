---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
permalink: /
title: StarkeBlog
---

# StarkeBlog

{% for post in site.posts limit:3 %}
   <article>
        <h2>
            <a href="{{ post.url }}">
                {{ post.title }}
            </a>
        </h2>
        <time datetime="{{ post.date | date: "%Y-%m-%d" }}">{{ post.date | date_to_long_string }}</time>
        {{ post.content }}
    </article>
{% endfor %}