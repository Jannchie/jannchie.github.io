---
title: Travis-CI深坑：如何解决Bad decrypt问题？
tags: [travis, 自动化, 构建]
categories: 有限深度的天坑
date: 2019-04-17
---

在windows下，使用travis-cli命令行工具遇到如下错误：

``` bash
bad decrypt

140356638541472:error:0606506D:digital envelope routines:EVP_DecryptFinal_ex:wrong final block length:evp_enc.c:532:
```

<!--more-->

在使用 Travis-CI 的过程中，我有这样一个需求，我需要在 travis 自动构建完毕后，构建结果上传到自己的服务器。为了完成这波操作，我需要让travis构建服务器能够登录我自己的服务器。而登录我的服务器需要登录凭证，如果本地构建的话，我可以自己输入密码，而travis的构建过程是无法交互的，也就说并不能输入密码。因此，我要使用[公钥认证](https://zh.wikipedia.org/wiki/%E5%85%AC%E5%BC%80%E5%AF%86%E9%92%A5%E5%8A%A0%E5%AF%86)的方式进行登录。

关于公钥认证的详细情报在这里就不细说了，总之，我需要将公钥发送给travis构建服务器，这样这个服务器就可以使用公钥加密内容，安全地传输给我自己的服务器。

然鹅，要让travis得知公钥，必须要把公钥放在代码仓库中，这个设计其实挺土的。如果把公钥直接放在代码仓库相当于把密码告诉所有人。因此，需要对这个公钥再进行加密。而travis-cli提供了[快速加密的方式](https://docs.travis-ci.com/user/encrypting-files)。

而根据上述教程，进行构建的话，就有可能会出现最开头出现的那个错误。我查阅了 travis cli 项目的 issue。发现别人也有[同样的问题](https://github.com/travis-ci/travis-ci/issues/4746)。

解决方案是：**不要使用Windows进行加密！**

总之，我使用Ubuntu子系统进行加密，就不会出现这个问题了。

Travis cli 和 Windows 之间，必有一个辣鸡，我倾向于 Travis 是辣鸡（逃。