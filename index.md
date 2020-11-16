---
layout: page
title: Hello World ARM64!
tagline: regexisart
---
{% include JB/setup %}

## Ready blogs 

<ul class="posts">
  {% for post in site.posts %}
    <p><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></p>
  {% endfor %}
</ul>

## Doing list

- ARM CI
- open source enhancement on ARM64

## Tips

```python
if __name__ == '__main__':
    print('Hello World ARM64!')
```

----------

陈锐
RuiChen
@kiwik at weibo.com
chenrui.momo@gmail.com
