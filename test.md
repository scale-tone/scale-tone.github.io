## Test

{% for post in site.posts %}
	{{ post.title }}  |[Click Here]({{ post.url }})
{% endfor %}
