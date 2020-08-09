---
title: "解决Spring boot @Async 注解无效的问题"
tags: [BiliOB观测者, 后端, Spring boot]
categories: 踏向征途的轨迹
date: 2020-02-23 23:15:10
---

当我调用@Async注解修饰的方法时，并没有按照期待异步执行，直接返回先行返回结果。

在[Stack Overflow](https://stackoverflow.com/questions/4060718/async-not-working-for-me)上找到了解决方案：

> 当我们从一个对象的某一方法中，调用同一对象的@Async方法时，我们会绕过异步代理代码而只是在同一个线程中调用普通方法。
>
> 解决这个问题的一种方法是确保对@Async方法的调用来自另一个对象。[David's Dev Notes](http://groovyjavathoughts.blogspot.com/2010/01/asynchronous-code-with-spring-3-simple.html)

一种强行异步使用当前对象中方法的代码如下：

> 我们可以在自己的类中调用异步方法，但必须创建一个自引用bean。这里惟一的副作用是我们将不能在构造函数中调用任何异步代码。这样我们可以让代码保持在同一位置。

``` java
@Autowired ApplicationContext appContext;
private MyAutowiredService self;

@PostConstruct
private void init() {
    self = appContext.getBean(MyAutowiredService.class);
}

public void doService() {
    //This will invoke the async proxy code
    self.doAsync();
}

@Async
public void doAsync() {
    //Async logic here...
}
```
