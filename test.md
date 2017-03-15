## Test

{% for post in site.posts %}

   - {{ post.date | date: "%Y-%m-%d" }} &nbsp;&nbsp;&nbsp; [{{ post.title }}]({{ post.url }})

{% endfor %}

<a href="https://www.linkedin.com/in/lepeshenkov">
   <img src="https://static.licdn.com/scds/common/u/img/webpromo/btn_myprofile_160x33.png" width="160" height="33" border="0" alt="View Konstantin Lepeshenkov's profile on LinkedIn">
</a>
