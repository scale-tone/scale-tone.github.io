## Test
{% for post in site.posts %}

   - {{ post.date | date: "%Y-%m-%d" }} &#9; [{{ post.title }}]({{ post.url }})

{% endfor %}
