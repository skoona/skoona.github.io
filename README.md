## Welcome to my Blog


I use this site to share my strategy on developing applications or components related to my current interests.

Ruby and Ruby on Rails are a part of my day to day development activities.

### Notable Shares
* Rails V5: [SknServices](https://skoona.github.io/SknServices/)
* * [SknServices Demo](http://vserv.skoona.net:8080/)
* Ruby Gem: [SknUtils](https://skoona.github.io/skn_utils/)
* C: [Gtk3 BarGraph Widget](https://skoona.github.io/glinegraph-cairo/)
* C: [Gtk3 Rpi Utilities](https://skoona.github.io/skn_rpi-display-services/)

I will be posting new articles that go in depth on the above strategy, here!

<div id="home">
  <h1>Blog Posts</h1>
  <ul class="posts">
  {% for post in site.posts %}
    <li>
      <span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
  {% endfor %}
  </ul>
</div>
