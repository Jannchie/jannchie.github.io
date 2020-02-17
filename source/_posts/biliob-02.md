---
title: "BiliOB开发日志 #02"
tags: [BiliOB观测者, 前端, 后端, Spring Boot]
categories: 踏向征途的轨迹
date: 2020-02-16 19:12:00
---

今天添加一些早就想加入的功能，一个是评论，一个是up主列表。

<!--more-->

## 修改个人信息 2020-02-16 12:37:09

个人信息页面的表单对齐太不美观。这是因为之前使用了Flex进行横向布局，然而这么操作不太科学，因为表单的`text field`和`button`的大小非常不合适。之所以这么做写代码是因为不太了解vuetify的组件功能。

在新的版本中，我使用了`append icon`的方式处理按钮，使得整体更为美观。

---

添加了对email的修改。mail的修改最好能够通过邮件验证下用户，然而这样做太麻烦了，暂且放下。

## 评论功能 2020-02-16 13:28:00

评论打算根据页面路径来存储。每个页面都可以拥有评论。

用户可以发布评论，也可以顶或者踩评论。

顶或者踩评论可以获取积分。

发现以前的获取积分功能可以重构。

之前的获取积分功能采用AOP切面实现，但是实现得比较丑陋，需要在一个固定类里按照固定的参数类型和顺序的编写代码：

``` java
// com/jannchie/biliob/credit/handle/CreditHandle.java
public ResponseEntity<Result<String>> modifyUserName(User user, CreditConstant creditConstant, String newUserName) {
  mongoTemplate.updateFirst(
    Query.query(Criteria.where("_id").is(user.getId())),
    Update.update("nickName", newUserName),
    "user");
  return getSuccessResponse(user, creditConstant, newUserName);
}
```

现在可以通过一个lambda函数来编写代码：

``` java
// com/jannchie/biliob/credit/handle/CreditHandle.java
public ResponseEntity<Result<String>> doCreditOperation(User user, CreditConstant creditConstant, Execution execution) {
  return getSuccessResponse(user, creditConstant, execution.execute());
}

@FunctionalInterface
public interface Execution {
  /**
    * 执行操作
    *
    * @return 并返回填充回执的文字
    */
  String execute();
}
```

这样可以在调用出编写相关操作：

``` java
// com/jannchie/biliob/service/impl/UserCommentServiceImpl.java
/**
 *  提交评论功能
 */
@Override
public ResponseEntity<Result<String>> postComment(Comment comment) {
  User user = LoginChecker.checkInfo();
  return creditHandle.doCreditOperation(user, CreditConstant.POST_COMMENT, () -> {
    comment.setDate(Calendar.getInstance().getTime());
    comment.setLikeList(new ArrayList<>());
    comment.setDisLikeList(new ArrayList<>());
    comment.setUserId(user.getId().toHexString());
    mongoTemplate.save(comment);
    return comment.getPath();
  });
}
```

之所以以前没有使用这种方式，单纯是因为对Java的lambda表达式不熟悉。使用了lambda表达式后，终于感觉科学了！
