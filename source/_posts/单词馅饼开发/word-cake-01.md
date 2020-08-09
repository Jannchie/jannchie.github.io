---
title: "单词馅饼开发日志 #01"
tags: [单词馅饼, 前端, 后端, Spring Boot, MongoDB]
categories: 踏向征途的轨迹
date: 2020-02-17 23:24:39
---

这是一款背单词的web。

最近要继续刷托福，因此做了这样一个APP，用来背诵单词。

<!-- more -->

![单词馅饼-个人中心页面](/images/word-cake-01-1.png "单词馅饼-个人中心页面")

## 2020-02-17 17:47:22

在处理动画上，遇到了很多麻烦。

Vue自带了Transition动画，但是非常不好用，根据一段时间的研究，发现只能完成进入、退出、换位的效果。如果要做针对dom属性的修改，也许可能实现，但是实现方式总觉得非常诡异。

本次遇到了的问题是，我希望能够动态设置App Tool Bar的各个组件。具体来说，我希望Bar上初始状态下具有一个搜索栏和一个标题，而单击搜索栏后，标题将消失，而搜索栏占据整个开头。

我本来觉得这会是一个非常简单的功能，但是实现起来却花了一番功夫。本来打算使用第三方库的，确实发现了一个看上去非常神奇的库，叫做[anime](https://animejs.com/)。不过用在我这个项目里实在是杀鸡用牛刀了。

于是决定使用CSS动画实现。

CSS动画需要其实用起来也不是很方便，要想使用CSS动画监听属性，必须先定义初始状态和结束状态：

``` html
<!-- 针对标题 -->
<v-toolbar-title
  :style="
    search.searching
      ? `max-width:0%; transition:all 0.2s;`
      : `max-width:100%; transition:all 0.3s;`
  "
  >{{ pageTitle }}
</v-toolbar-title>
<!-- 针对搜索栏 -->
<v-text-field
  id="search-field"
  v-if="$route.path.indexOf('recite') == -1"
  solo
  :style="
    search.searching
      ? `max-width:100% ;transition:all 0.2s; margin-left: auto;`
      : `max-width:150px ;transition:all 0.3s; margin-left: auto;`
  "
  outlined
  v-model="search.searchText"
  @focus.stop="doSearch"
  dense
  rounded
  flat
  hide-details
  label="搜索"
  :prepend-inner-icon="search.searching ? `mdi-arrow-left` : `mdi-magnify`"
  @click:prepend-inner="hiddenSearch"
/>
```

注意到我使用了`max-width: 0%`来虚假地隐藏标题。这样能够更容易做成流畅的动画。这么做使得标题不能拥有`margin`属性，因为`margin`仍然会占据空间。

于是我对搜索框使用了`margin-left: auto;`来使得搜索框能够靠右浮动。

## 2020-02-17 20:36:25

搜索功能仿造知乎，不是直接进行搜索，而是弹出一个独立的搜索页面。

但是这样产生了很严重的滚动穿透问题，也就是说滑动搜索框，会同时移动背景板，且同时如果背景的页面很长，搜索框页面会整体变长，这样非常不优雅。但是目前没有特别好的解决方法，先放在一边等会再做。

## 2020-02-17 23:01:34

之前迁移本地MongoDB到阿里云服务器上，并没有迁移索引数据，导致索引无效，无法使用MongoDB的全文索引，于是我对索引进行重建。

从[MongoDB官方文档](https://docs.mongodb.com/manual/core/index-text/#index-feature-text)发现了一种更加方便的建立全文索引的方式：

> `db.collection.createIndex( { a: 1, "$**": "text" } )`

这样可以自动对全部字段进行全文索引。

不过这样建立索引非常慢，毕竟我需要建立索引的单词数据库相对较为庞大。

于是加上参数：

``` js
db.word.createIndex( { a: 1, "$**": "text" }, { background: true } )
```

这样一来，便可以后台进行索引建立，总共花费大约2000秒的时间。

搜索方面，为了能够复用已经编写完毕的组件，决定全盘使用`ReciteRecord`来包裹`Word`

---

## 2020-02-17 23:24:21

独立的搜索页面并不好搞，准备择日重构！
