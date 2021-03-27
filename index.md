My name is Konstantin Lepeshenkov, I'm currently acting as a Cloud Solution Architect @ Microsoft, and here are my personal thoughts.

{% for post in site.posts %}

   - {{ post.date | date: "%Y-%m-%d" }} &nbsp;&nbsp;&nbsp; [{{ post.title }}]({{ post.url }})

{% endfor %}
