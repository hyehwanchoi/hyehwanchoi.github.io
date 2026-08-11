---
layout: archive
title: "최신 글"
permalink: /latest/
author_profile: true
---

<p class="latest-posts__intro">최근에 기록한 글부터 차례로 모았습니다. 운영 중 마주친 문제와 해결 과정을 편안하게 읽어보세요.</p>

<div class="latest-posts" aria-label="최신 글 목록">
  {% for post in site.posts %}
    {% include archive-single.html %}
  {% endfor %}
</div>
