---
title: "BiliOB开发日志 #03"
tags: [BiliOB观测者, 前端]
categories: 踏向征途的轨迹
date: 2020-02-19 00:23:22
---

今天要实现评论功能。

具体计划是这样：

- **PC端**：增宽容器宽度，在每一个页面的右侧加一栏评论栏。
- **手机端**：直接在底部添加评论栏。

<!--more-->

## 后端修改 2020-02-17 23:42:59

编写完基本框架后，发现前端请求不到用户的详细信息。

因此在Comment中加入一个新的字段User，修改获取评论的方法，通过一次lookup + unwind操作获取唯一的用户（发布者）：

``` java
@Override
public List<Comment> listComments(String path, Integer page, Integer pageSize) {
  AggregationResults<Comment> ar = mongoTemplate.aggregate(
    Aggregation.newAggregation(
      Aggregation.match(Criteria.where("path").is(path)),
      Aggregation.lookup("user", "userId", "_id", "user"),
      Aggregation.unwind("user"),
      Aggregation.project().andExpression("{ password: 0, favoriteMid: 0, favoriteAid: 0 }").as("user"),
      Aggregation.skip(page * pageSize),
      Aggregation.limit(pageSize)
    ), Comment.class, Comment.class);
  return ar.getMappedResults();
}
```

这么操作了之后，发现无法请求到数据了。

经过调试发现，并没有办法使用$lookup根据userId作为外键寻找user中的记录。

原因可能是Comment中，userId保存的是String类型的数据，而User的_id字段类型是ObjectId。虽然使用find可以自动将String转换为ObjectId，但是在aggregation中似乎不可以。

搜索了一番之后，感觉并不好手动格式转换，因此只能修改model，将Comment中的userId字段修改为ObjectId类型。

**后端的数据还是以ObjectId类型存储较好。**之所以之前没有这么做，是因为有时前端需要id字符串用于查找。要解决这类需求，一种方式是前端手动转换。

但是发现了一种更为科学的方式，可以在后端Java中，直接把@Id注解修饰的字段类型设置为String即可。Spring data mongo 可以自动地转换。

---

## 前端布局 2020-02-18 12:27:58

前端经过一番修改，主体部分偏向了一边，我希望主体仍然在中央部分。

通过在对称侧添加一个`<v-spacer></v-spacer>`并设定宽度与评论组件相同即可完成这一需求。

## 评论组件滚动 2020-02-18 22:55:31

我希望评论组件能够滚动，并且它的高度不超过主体组件。通过设定评论组件的CSS可以搞出滚动条：

``` css
#comment-container {
  overflow-y: auto;
  /* 为了能够搞出滚动条，我们需要设定他的高度 */
  max-height: 1000px
}
```

然而，主体组件的高度是不确定的，所以我们不能直接定死`max-height`属性。

也就是说，我们需要设定评论组件的高度，使之等于主体组件的高度。

如何根据某一元素的高度自动设定自身的高度呢？

综合查询后，发现还是使用定时任务`setInterval`来实现最为简单。如果有更合适的方案，以后再来重构。

mounted中添加下列代码：

``` js
// 如果在大屏下，则定时每秒同步大小变化
if (this.$vuetify.breakpoint.lgAndUp) {
  setInterval(this.resize, 1000);
}
```

在methods中添加同步函数：

``` js
// 同步大小变化
resize() {
  document.getElementById("comment-container").style.maxHeight =
    document.getElementById("main-view").offsetHeight + "px";
}
```

## 滚动样式修改 2020-02-18 23:52:07

自带的滚动条样式太丑陋了。

从[滚动条CSS样式参考](https://codepen.io/GhostRider/pen/GHaFw)，我找到了一个漂亮的滚动条样式，稍作修改后用于本网站。
