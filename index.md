## Hey, it's me, Konstantin Lepeshenkov

Former software architect at Kaspersky Lab, I currently work as a Cloud Solution Architect for Microsoft.

Here I'm talking about my open-source project(s), clouds and some other dev stuff. Stay tuned.

{% for post in site.posts %}

   - {{ post.date | date: "%Y-%m-%d" }} &nbsp;&nbsp;&nbsp; [{{ post.title }}]({{ post.url }})

{% endfor %}
