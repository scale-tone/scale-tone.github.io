## Test

{% for post in site.posts %}
	{{ post.date | date: "%B %e, %Y" }} [test](http://github.com)
{% endfor %}