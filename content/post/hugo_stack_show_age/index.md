---
title: Hugo+Stack主题显示博客运行时间
description: 
slug: hugo_stack_show_age
date: 2024-04-20 00:00:00+0000
image: 
categories:
    - 杂项
tags:
    - Hugo Stack
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

## 参考博客

大神博客底部js: https://icloudnative.io/

## 网站引入js

将js添加到文件`layouts\partials\footer\custom.html`中。
原理就是计算当前时间-博客的生日，然后将计算结果渲染到对应的标签中。函数每隔1s执行一次。

``` js
<script defer language="javascript">
    var uptime_1 = "本站已经开心运行 ";
    var uptime_2 = " 天 ";
    var uptime_3 = " 小时 ";
    var uptime_4 = " 分 ";
    var uptime_5 = " 秒";

    function show_date_time() {
        window.setTimeout(show_date_time, 1e3);
        BirthDay = new Date("8/4/2024 22:54:23");
        today = new Date();
        timeold = today.getTime() - BirthDay.getTime();
        sectimeold = timeold / 1e3;
        secondsold = Math.floor(sectimeold);
        msPerDay = 24 * 60 * 60 * 1e3;
        e_daysold = timeold / msPerDay;
        daysold = Math.floor(e_daysold);
        e_hrsold = (e_daysold - daysold) * 24;
        hrsold = Math.floor(e_hrsold);
        e_minsold = (e_hrsold - hrsold) * 60;
        minsold = Math.floor((e_hrsold - hrsold) * 60);
        seconds = Math.floor((e_minsold - minsold) * 60);
        document.getElementById('span_show_age').innerHTML = uptime_1 + daysold + uptime_2 + hrsold + uptime_3 + minsold + uptime_4 + seconds + uptime_5;
    }

    show_date_time();
</script>
```


## 最终效果




