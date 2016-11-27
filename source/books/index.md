title: 悦读•时光
date: 2016-11-26 11:03:29
comment: true
---
记录一下看过的感觉不错的书。
### 最近阅读
{% stream %}
<!-- {% figure  []() %} -->
{% figure http://blog.xujin.org/css/book/jvm.png [jvm](#) %}
{% figure http://blog.xujin.org/css/book/bf.png [并发编程](#) %}
{% figure http://blog.xujin.org/css/book/wfw.png [微服务](#) %}
{% figure http://blog.xujin.org/css/book/netty.png [Netty](#) %}
{% endstream %}
### 历史阅读
{% stream %}
<!-- {% figure  []() %} -->
{% figure http://blog.xujin.org/css/book/1.png [RabbitMQ](#) %}
{% figure http://blog.xujin.org/css/book/3.png [Activiti](#) %}
{% figure http://blog.xujin.org/css/book/4.png [docker](#) %}
{% figure http://blog.xujin.org/css/book/5.png [软件工程](#) %}
{% figure http://blog.xujin.org/css/book/6.png [Spring源码分析](#) %}
{% figure http://blog.xujin.org/css/book/go.png Go](#) %}
{% figure http://blog.xujin.org/css/book/gradle.png [gradle](#) %}
{% figure http://blog.xujin.org/css/book/maven.png [maven](#) %}
{% figure http://blog.xujin.org/css/book/os.png [OpenStack](#) %}
{% figure http://blog.xujin.org/css/book/spring-data.png [spring-data](#) %}
{% endstream %}
---

<script type="text/javascript">
$(function() {
  var shown = 18;
  $('figure').slice(shown).addClass('foldable');
  if ($('.foldable').length > 0) {
    $('.foldable').hide();
    $('div.hexo-img-stream').after('<p id="btn-more" class="article-more-link" align="center"><a href="javascript:;">Show More</a></p>');
    $('#btn-more').click(function() {
      $('.foldable').show();
      $('#btn-more').hide();
    });
  }
});
</script>

