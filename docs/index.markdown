---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
title: teacurve
---


## Recent Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>


## Find posts

- [Posts by Tag]({{ site.baseurl }}{% link tags.markdown %})

- [Posts by Category]({{ site.baseurl }}{% link categories.markdown %})