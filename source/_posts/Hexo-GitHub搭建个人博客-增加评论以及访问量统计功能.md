---
title: Hexo GitHub搭建个人博客-增加评论以及访问量统计功能
date: 2016-07-11 18:52:47
tags: 开发工具
---
# 内容概要

本文主要介绍如何为Hexo搭建的博客增加评论功能，采用了多说插件，以及访问量统计功能。

<!-- more -->

# 增加评论功能的步骤

## 注册多说

进入多说官网，点击我要安装，注册账号。填入多说域名（后续会用到）、要使用多说插件的站点地址等信息。

## 修改hexo配置文件

- 在根目录下_config.yml中增加如下内容：

```
duoshuo_shortname: 你站点的short_name(xiajun)
```

这里的short_name也就是前面注册多说填入的多说域名。

- 然后将themes\landscape\layout_partial\article.ejs（如果使用的其他主题，文件路径类似）中的如下内容：

```
<% if (!index && post.comments && config.disqus_shortname){ %>

<section id="comments">
  <div id="disqus_thread">
    <noscript>Please enable JavaScript to view the <a href="//disqus.com/?ref_noscript">comments powered by Disqus.</a></noscript>
  </div>
</section>
<% } %>
```

替换为：

```
<% if (!index && post.comments && config.duoshuo_shortname){ %>

  <section id="comments">
  <!-- 多说评论框 start -->
<div id="ds-thread" class="ds-thread" data-thread-key="<%= post.path %>" data-title="<%= post.title %>" data-url="<%= post.permalink %>"></div>
<!-- 多说评论框 end -->
<!-- 多说公共JS代码 start (一个网页只需插入一次) -->
<script type="text/javascript">
var duoshuoQuery = {short_name:"xiajun"};
  (function() {
          var ds = document.createElement('script');
              ds.type = 'text/javascript';ds.async = true;
            ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.js';
              ds.charset = 'UTF-8';
          (document.getElementsByTagName('head')[0]
              || document.getElementsByTagName('body')[0]).appendChild(ds);
  })();
  </script>
<!-- 多说公共JS代码 end -->
  </section>
  <% } %>
  
```

然后将修改部署到github。这个时候刷新博文，就可以发现文章下面多出来的评论窗口。
发现评论内容会自动刷新，但是评论数量不会更新，不知道是否是多说的一个bug。

# 增加访问量统计功能

- 修改themes/landscape/layout/_partial/after_footer.ejs文件，引入js文件

```
<script async src="https://dn-lbstatics.qbox.me/busuanzi/2.3/busuanzi.pure.mini.js">
</script>
```

- 然后在我们要展示访问量统计信息的地方添加如下代码

```
本站总访问量<span id="busuanzi_value_site_pv"></span>次
本站访客数<span id="busuanzi_value_site_uv"></span>人次
本文总阅读量<span id="busuanzi_value_page_pv"></span>次
```

部署到github，这个时候就发现对应的地方展示出来了访问量统计信息：

# 参考链接
- [在hexo中加入多说评论](http://www.lichanglin.cn/%E5%9C%A8hexo%E4%B8%AD%E5%8A%A0%E5%85%A5%E5%A4%9A%E8%AF%B4%E8%AF%84%E8%AE%BA/)
