## Konstantin Lepeshenkov's blog

I'm a Principal Dev @ Microsoft, and here are my personal thoughts 👇

{% for post in site.posts %}

   - {{ post.date | date: "%Y-%m-%d" }} &nbsp;&nbsp;&nbsp; [{{ post.title }}]({{ post.url }})

{% endfor %}
