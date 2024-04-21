---
title: Hugo+Stack主题集成Vercount统计网站访问信息
description: 
slug: hugo_stack_vercount
date: 2024-04-20 00:00:00+0000
image: 
categories:
    - Blog
tags:
    - Hugo Stack
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 参考文章

https://stack.jimmycai.com/config/header-footer

https://stack.jimmycai.com/config/footer

[umami](https://umami.is)


[Vercount官网](https://vercount.one/)

![](https://s3.bmp.ovh/imgs/2024/04/20/cf50449037262da6.png)

**推荐使用Vercount，不推荐使用不蒜子。理由：不蒜子网站太老了，作者没有维护的意思，Vercount官网更现代化，开源，点开看看就知道了。我集成的时候官网才几百的访客。看GitHub提交记录，也就最近3个月刚成立的项目，很年轻，有活力。**

## 集成Vercount

我使用的是Hugo+Stack主题，根据Stack官方文档，在主题根目录下，新建一个自定义文件`layouts\partials\footer\custom.html`。

### 网站引入vercount的js

然后将Vercount的js添加到该文件中。

``` js
<script defer src="https://cn.vercount.one/js"></script>
```

### 网站底部加入统计整个站的信息

根据Stack文档，自定义网站的footer信息。在`param.toml`文件中的`customText`加入以下标签。

``` toml
[footer]
customText = """网站总访客数：<span id='busuanzi_value_site_uv'>Loading</span><br/>
      网站总访问量：<span id='busuanzi_value_site_pv'>Loading</span>
     """
since = 2024
```

### 博客页面加入统计当前博客的信息

在博客详情页面`layouts\partials\article\components\footer.html`,添加统计博客的访问情况。

``` html
<footer class="article-footer">
    // 略
    <section>
        页面浏览量<span id="busuanzi_value_page_pv">Loading</span>
    </section>
</footer>
```

## 集成Umami
https://umami.is/

**Umami是一个网站分析工具，它可以分析出一个网站的详细访问数据，包括请求PV、UV、国家来源、来源于哪个网站、用户的操作系统、浏览器等等。**
注册账号，添加网站成功后，开启共享URL，然后将这个url放到自己的网站即可。
参考这个链接启用。 https://umami.is/docs/enable-share-url

``` toml
[footer]
customText = """网站总访客数：<span id='busuanzi_value_site_uv' style='margin-right:50px'>Loading</span> 网站总访问量：<span id='busuanzi_value_site_pv' style='margin-right:50px'>Loading</span>
<a href='https://eu.umami.is/share/y6kXo4CE3oYHXQ37/farb.github.io' style='color:blue' target='_blank'>网站访问详细统计</a>
<br/>
      <span id='span_show_age'  style='margin-right:50px'/>
     """
```

## 最终效果

Vercount集成效果如下：

![](https://s3.bmp.ovh/imgs/2024/04/20/7bf7133a95813d68.png)

Umami集成效果如下：
![](https://s3.bmp.ovh/imgs/2024/04/20/02e70ec3e5aa8aab.png)
