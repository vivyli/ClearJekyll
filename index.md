---
layout: home
title: Notebook
tagline: Supporting tagline
---
{% include JB/setup %}


<ul class="posts">
  {% for post in site.posts %}
    <!--li>
    <!span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a>
    
    </li-->
    <div class="post">
        <h3 class="ntitle">
        <a href="{{ BASE_PATH }}{{ post.url }}" class="ntitle">{{ post.title }}</a>
        <span class="date"> {{ post.date | date_to_string }}</span>
        </h3>

        <span> {{ post.content  | split:'<!-- more -->' | first }}</span>
        <span class="readmore"><a href="{{ post.url }}"> &gt;&gt; Read More</a></span>
     

    </div>
  {% endfor %}
</ul>


