---
title: Java下的异常处理
date: 2018-11-27 15:57:12
tags: [后端,Java,Spring Boot]
categories: 形而上的最佳实践
---

我在一个项目中用到了Spring Boot，需要处理一系列的业务。业务如果正常执行则不必多说，倘若出现一些意外的情况：比如说创建一个用户但是用户已经存在，比如说本该输入一串数字但却接收到一串字符，又比如说本该从数据库里取出一条数据但是却什么都没有得到……在这些情况下，到底是应该抛出一个自定义的Expection还是使用一系列的Else if语句之类的对这些意外的情况进行判断呢？Expection下又有很多子类，自定义异常到底应该继承RuntimeException还是IOException呢？虽然实际使用中无论哪种方式都能解决问题，但到底有没有一个最佳实践？这就是本篇文章讨论的问题。

<!--more-->

## 先回顾一下基础知识

Java标准库中内置了通用异常类，这些异常类都继承自Throwable，含义很明显，即“可抛出的类”。Throwable又派生出Error类和Exception类。Error类代表错误：

> **错误**： 错误不是异常，而是脱离程序员控制的问题。错误在代码中通常被忽略。例如，当栈溢出时，一个错误就发生了，它们在编译也检查不到的。
> 
> <div style="text-align: right"> ——菜鸟教程 </div>

错误并非我们需要关注的内容，Java 程序通常不捕获错误。错误一般发生在严重故障时，它们在Java程序处理的范畴之外。而作为一个程序员，我们更关注的是是程序本身可以处理的Exception。

Exception又派生出RuntimeException（运行时异常）和IOException（IO异常），这两者最大的区别在于：RuntimeException和Error同属于checked exceptions（已检查的异常）而IOException属于unchecked exceptions（未检查的异常）。具体的解释如下：

> **未检查的异常**： 未检查的异常是可能被程序员避免的异常。与已检查的异常相反，运行时异常可以在编译时被忽略。
> 
> **已检查的异常**： 用户错误或问题引起的异常，这是程序员无法预见的。例如要打开一个不存在文件时，一个异常就发生了，这些异常在编译时不能被简单地忽略。
> 
> <div style="text-align: right"> ——菜鸟教程 </div>

以上就是异常相关的一些概念。光看这些概念其实依旧很难理解，其实本来我并没有思考他们之间的不同，只有在项目中真正使用的时候才能感觉到：

当你在程序中抛出一个未检查的异常，那么他的调用者不必处理之。而如果抛出一个已检查的异常，那么调用者必须处理他，或者接着往上抛，直到他被真正地处理。

但这只是他们的表现形式而已，实际上未检查异常和已检查异常之间的界限颇不明确，为什么打开一个文件失败就算是“已检查异常”，而下标越界就算是“未检查异常”？

## 无需使用Exception？

一开始我的设计是这样的：我希望能够预先定义出程序可能出现的各种异常。比如说用户不存在异常，比如说密码错误异常，并且这些异常都继承自Exception。在发生这些错误的时候统一抛出异常，比如说下面的验证用户身份功能：

``` java
/**
 * 获取身份验证信息，采用Shiro实现。
 * 这个方法覆写Shiro中doGetAuthenticationInfo方法。
 *
 * @param authenticationToken 用户身份信息 token
 * @return 返回封装了用户信息的 AuthenticationInfo 实例
 * @throws AuthenticationException wrong password exception
 */
@Override
protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationTokenauthenticationToken)
    throws AuthenticationException {
  UsernamePasswordToken token = (UsernamePasswordToken) authenticationToken;
  // 从数据库获取对应用户名密码的用户
  String password = userService.getPassword(token.getUsername());
  if (!password.equals(new String((char[]) token.getCredentials()))) {
    throw new AccountException("密码不正确");
  }
  logger.info(token.getUsername());
  return new SimpleAuthenticationInfo(token.getPrincipal(), password, getName());
}
```

然后运用@controllerAdvice注解，在一个handler中统一地处理这些异常，处理的方式比较统一，即返回错误码和错误信息：

``` java
@RestControllerAdvice
public class AccountExceptionHandler {
  private static final Logger logger = LogManager.getLogger(AccountExceptionHandler.class);
  @ResponseBody
  @ExceptionHandler(AccountException.class)
  @ResponseStatus(value = HttpStatus.FORBIDDEN)
  public ExceptionResult handleAccountException(AccountException ex) {
    String msg = ex.getMessage();
    // 生成返回结果
    ExceptionResult errorResult = new ExceptionResult();
    errorResult.setCode(403);
    errorResult.setMsg(msg);
    logger.info(msg);
    return errorResult;
  }
}
```

这样处理看上去不错，但是真的科学吗？

在经过一番搜索后，我发现这并不是Exception的正确用法。实际上，Exception，也就是已检查异常，在许多语言中（比如优雅的C#或者kotlin中）已经被抛弃了。所以在需要使用异常的时候我们都可以使用RuntimeException。已检查异常并不是没有用的，当您希望您的API的用户考虑如何处理异常情况(如果它是可恢复的)时，可以使用已检查异常。只是在Java平台中检查过的异常被过度使用，这使得人们讨厌它们。[^1]

国内也有一些类似的观点，比如：

> Exception 本身一个非常好的机制，只在可以处理地方catch下来，避免了C语言那种层层 if 判断向上返回错误的繁琐。然而，绝大多数商业应用场景中，唯一合理的处理方式就是汇报异常并放弃执行。既然handle的代码一样，那么 catch(Exception) 就自然成为绝大多数程序员的选择。这可能是引入 Exception 机制之初所始料未及的。鉴于这样的事实，CE的必要性就值得怀疑了。
> <div style="text-align: right"> ——by Michael Li[^2] </div>

事实上，我们的异常不需要太多区别处理，只是需要包装一下返回一条错误信息而已，在这种情况下，似乎没有什么显式声明Exception并处理的必要了。

## 无需使用RuntimeException？

于是我修改代码，使用RuntimeException来代替Exception。这样确实非常轻松，不用再写大量的throw了。然而在这个时候我又看到了另一种说法，他们认为RuntimeException是一种不应该发生的情况，需要用编程手段（比如else if语句)避免的一种情况：

> Runtime exception serve a specific purpose - they signal programming problems that can be fixed only by changing code, as opposed to changing the environment in which the program runs. 
> <div style="text-align: right"> ——by dasblinkenlight[^3] </div>

如果按照这种理论的话，诸如在试图登录的时候却密码错误的这种情况，应该是一种需要被考虑到的可能性，而不是所谓的异常。另一个回答者讲得更为易懂：

> Because they're things that will happen normally. Exceptions are not control flow mechanisms. Users often get passwords wrong, it's not an exceptional case. Exceptions should be a truly rare thing, UserHasDiedAtKeyboard type situations.
> <div style="text-align: right"> ——by blowdart[^4] </div>

这位回答者认为，只有用户死在了键盘上这种很少出现的情况，才是异常。为什么要这样？因为异常是有开销的，事实上异常包括了出现异常的全部调用栈信息，而传输这些信息需要多余的时间和内存。所以在实际项目运用中应该尽可能的避免。

按照这种说法，如果说Exception是必须显式处理的异常，那么RuntimeException就是处理不了的异常，应该在报出RuntimeException之前通过编程的处理掉，实在没兜住才报RuntimeException，事后再修复这个设计错误。

但实际上大家不是这么做的，事实上大家都用RuntimeException当做控制语句，正常业务流程是一套逻辑，异常处理直接跳转到handler里是一套异常处理逻辑。这种做法的好处很明显，就是逻辑清晰。缺点是有更多的内存和时间的开销。国内也有人认为这种开销是九牛一毛，我并没有时间和精力去做相关的测试，所以目前可能会维持原状。

## 需要自己定义异常吗？

我在项目中自己定义了许多异常，比如用户不存在（UserNotExistException）、用户已注册（UserAlreadyExistException）等等。但实际上是否有这个必要呢？有些人只定义一个BusinessException，预先定义好枚举的一组异常类型，抛出新建的异常时直接在构造函数传入异常的类型就好。现在看来这种解决方案更加合适。因为对大部分业务异常而言，能做的也只有返回一个错误码和错误信息了。这样能消灭掉大部分重复的代码，极大地增强可读性。当然如果要对异常做一些特殊的处理的话还是需要自己另行定义的。

## 对于已检查异常，需要抛出吗？

已检查异常在抛出后一定要显式声明，在我看来，异常最佳处理的地点就是异常发生的地方。虽然可能会在业务逻辑代码中混入异常处理的代码块，但如果不这么做而是抛出异常的话会造成数据的流动，平白多出内存和时间的开销。况且一般异常发生在service层，如果在这里发生异常的地方都无法处理，再上传到control层去处理感觉更不合适。抛出机制只是提供一种不处理异常的选择，而不代表鼓励使用。

## 总结

总体来说，我认为Java异常处理机制的最佳实践是：

- 定义一个BusinessException，可自定义异常类型，处理通用异常。这个异常类继承自RuntimeException，无需显式在方法上注明需要抛出。
- 如果对异常需要进行别的操作，那么这个异常需要继承自Exception，并且如果可能的话，原地处理异常。

[^3]: https://stackoverflow.com/questions/27538706/when-to-throw-runtime-exception

[^4]: https://stackoverflow.com/questions/77127/when-to-throw-an-exception?rq=1

[^1]: https://stackoverflow.com/questions/6115896/java-checked-vs-unchecked-exception-explanation

[^2]: https://www.zhihu.com/question/60240474/answer/173856389