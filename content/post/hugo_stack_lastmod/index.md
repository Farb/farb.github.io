---
title: Hugo+Stack主题修改最后更新时间位置和时间格式
description: 
slug: hugo_stack_lastmod
date: 2024-04-19 00:00:00+0000
image: 
categories:
    - Blog
tags:
    - Hugo Stack
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 参考文章

https://shitao5.org/posts/hugo-stack/

https://stack.jimmycai.com/config/header-footer

## 时间格式修改
在 param.toml文件中修改格式如下：

``` toml
[dateFormat]
published = "2006-01-02"  # 发布日期格式
lastUpdated = "2006-01-02 15:04:05" # 最后修改时间格式，一定要对应go的日期格式
```

## 位置修改

参考官方文档，在博客项目下创建一个目录`layouts\partials\article\components`，一定要和stack主题的路径对应上，思想就是hugo生成静态页面时，使用自定义的页面替换stack主题的页面。

details.html页面

```  html
<div class="article-details">
    // 略...
    <footer class="article-time">
        // 发布日期代码块
        {{ if $showDate }}
            <div>
                {{ partial "helper/icon" "date" }}
                <time class="article-time--published">
                    {{- .Date.Format (or .Site.Params.dateFormat.published "Jan 02, 2006") -}}
                </time>
            </div>
        {{ end }}

        // 阅读时间代码块
        {{ if $showReadingTime }}
            <div>
                {{ partial "helper/icon" "clock" }}
                <time class="article-time--reading">
                    {{ T "article.readingTime" .ReadingTime }}
                </time>
            </div>
        {{ end }}

        // 剪切到这里，并修改标签和上面的代码块一致
        {{ if ne .Lastmod .Date }}
        <div class="article-lastmod">
            {{ partial "helper/icon" "clock" }}
            <time>
                {{ T "article.lastUpdatedOn" }} {{ .Lastmod.Format ( or .Site.Params.dateFormat.lastUpdated "Jan 02, 2006 15:04 MST" ) }}
            </time>
        </div>
        {{- end -}}
    </footer>
    {{ end }}

    // 略...
</div>
```

footer.html页面

```  html
<footer class="article-footer">
    // 略...

    // 这一块是默认的最后更新时间代码块，将这块代码剪切到details.html的正确位置
    {{- if ne .Lastmod .Date -}}
    <section class="article-lastmod">
        {{ partial "helper/icon" "clock" }}
        <span>
            {{ T "article.lastUpdatedOn" }} {{ .Lastmod.Format ( or .Site.Params.dateFormat.lastUpdated "Jan 02, 2006 15:04 MST" ) }}
        </span>
    </section>
    {{- end -}}

</footer>
```


