---
title: "BiliOB开发日志 #01"
tags: [BiliOB观测者, 前端, SEO]
categories: 踏向征途的轨迹
date: 2020-02-16 11:49:48
---

我决定在编程的同时，对编程时遇到的问题进行同步的思考和记录，这样更能节省时间，免除了事后记录的痛苦之感。

不知为何，BiliOB的Alexa排名一直在降低，最近甚至排出了榜外。于是决定进行一次SEO优化。我使用[woorank](www.woorank.com)对网站的SEO进行检测。

<!--more-->

## HTML标题标签添加 2020-02-15 14:10:25

第一步要做的是，添加HTML标题。

正常来说，HTML标题需要使用`<h1>`~`<h5>`这样的标签来包围起来，每个页面有且仅有一个一级标题标签。而我的网站没有使用这些标签，可能会影响爬虫对页面内容的判断。

页面的最上方有“BiliOB观测者”的字样，这是整个网站的标题，我将其设置为一级标题，为了保证样式不发生变化，我们将他的class设为title。这是vuetify的内置样式：

``` xml
<!-- src\components\layout\Master.vue -->
<VToolbarTitle> <h1 class="title">BiliOB观测者</h1></VToolbarTitle>
```

对于管理系统，也是同样的操作：

``` xml
<!-- src\components\Tracer\layout\Nav.vue -->
<VListItemTitle class="white--text">
  <h1 class="title">BiliOB观测者</h1>
</VListItemTitle>
<VListItemSubtitle :v-model="drawer">
  管理系统
</VListItemSubtitle>
```

对于每一个图表卡片的title，应用三级标题：

``` xml
<!-- src\components\biliob\Card.vue -->
<h3 v-else class="font-weight-light px-5 py-1">
  {{ title }}
</h3>
```

这样，就基本完成了标题的修改。

## 域名解析重定向 2020-02-15 14:48:01

下面一个问题比较严重，该网站目前能够通过`www`和`@`(不加`www`)两种域名访问。这对于一般用户方便的，有些人确实不喜欢加`www`。

合理的操作是，将访问`@`域的用户重定向到`www`。而当下，我这边会将这两种访问方式判断为不同的两个网站，从而可能无法准确记录流量数据，这很可能是导致排名下降的罪魁祸首。

解决方法是，登陆CDN管理平台，将原先的`@`记录删除，添加一条302跳转记录：

| 主机记录 | 记录类型 | 解析线路 | 记录值                 | TTL     | 状态 |
| -------- | -------- | -------- | ---------------------- | ------- | ---- |
| @        | 显性URL  | 默认     | https://www.biliob.com | 10 分钟 | 正常 |

解析修改完毕之后，针对 `http://` 的请求确实能够重定向了，但是 `https://` 协议的请求直接失败。

经过试验，发现在设置了显性URL跳转的情况下，结果为:

| 访问URL                 | 跳转结果                |
| ----------------------- | ----------------------- |
| http://biliob.com/      | https://www.biliob.com/ |
| http://www.biliob.com/  | https://www.biliob.com/ |
| https://biliob.com/     | TIMEDOUT                |
| https://www.biliob.com/ | https://www.biliob.com/ |

而未设置显性URL跳转时，结果为：

| 访问URL                 | 跳转结果                |
| ----------------------- | ----------------------- |
| http://biliob.com/      | NOTFOUND                |
| http://www.biliob.com/  | https://www.biliob.com/ |
| https://biliob.com/     | NOTFOUND                |
| https://www.biliob.com/ | https://www.biliob.com/ |

最后发现还是需要添加针对@的CNAME请求头，才能解决重定向问题：

| 访问URL                 | 跳转结果                |
| ----------------------- | ----------------------- |
| http://biliob.com/      | https://www.biliob.com/ |
| http://www.biliob.com/  | https://www.biliob.com/ |
| https://biliob.com/     | https://www.biliob.com/ |
| https://www.biliob.com/ | https://www.biliob.com/ |

但这样只能保证一定地区之内的可访问性。因为该网站同时还设置了MX解析——邮件确认功能。而MX解析与CNAME解析冲突。

目前还没有研究出比较好的解决方案。

---

## 使用localstorage代替cookies来控制本地版本 2020-02-15 22:18:57

chrome 80 对cookie策略进行了修改，现在需要添加SameSite来限制第三方Cookie。

而BiliOB使用了cookie来保存本地版本号，之前使用了cookies技术来进行保存，但其实没有必要，版本号并不需要在请求时上传到服务器，更好的策略是使用localstorage。

## 修改管理员邮箱 2020-02-16 02:55:28

针对上面的问题，一种解决方案是修改MX解析的邮箱。将`@`主机名修改为二级域名，比如`admin`。这样邮箱的格式会变成`xxx@admin.biliob.com`。虽然看上去长了一些，但是也同样非常正式。然后去除掉刚刚添加的显性跳转。就能完成全链路统一URL。
