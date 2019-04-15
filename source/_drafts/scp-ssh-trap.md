---
title: SCP、SSH深坑：如何解决无法免密登录问题？
tags: [travis, 自动化, 构建]
categories: 形而上的最佳实践
date: 2019-04-15
---

之前一直使用的是Jenkins。Jenkins是一个更加方便的自动化持续集成工具。之所以抛弃它是因为由于我本人没有闲置的服务器，因此这玩意必须要本地部署。考虑到不久的未来会更换电脑，重新部署Jenkins也比较麻烦；再加上不久之前，我本地的Jenkins由于未知原因出现崩溃，不断地卡我硬盘，花了很长时间才找到出现卡硬盘问题的原因......综上，我决定逐步取消使用本地Jenkins。由于我的代码大多都开源在了GitHub上，而Travis对GitHub的支持最好（实际上是只支持GitHub）。于是我决定使用Travis-CI对代码进行持续集成。

<!--more-->

---

Travis的大部分配置需要写在项目根目录下的`.travis.yml`文件里。每次对配置文件进行测试需要重新提交到GitHub。调试起来其实非常非常麻烦，而且会造成很多无用的提交记录。因此决定从`master`分支上派生一个新的`travis`分支。在调试完travis之后合并掉所有提交记录，再合并到`master`分支上，会是一个更加好的实践方式。

---

首先处理的是前端项目，前端项目基于VUE，自然基于的环境是node.js。前端没有太多需要保密的环境变量，因此配置非常简单：

``` yaml
language: node_js
node_js:
  - "8"
cache:
  directories:
    - node_modules
install:
  - npm install
script:
  - npm run build
branches:
  only:
    - travis
```

其中cache缓存node_modules文件夹可以大大提升构建速度。安装依赖只需要`npm install`一条命令，而编译也只需要`npm run build`一条命令即可。

测试环境下，构建的分支限定为`travis`分支。事实上目前也只有`travis`分支下有`.travis.yml`文件，因此也只能构建此分支。

<!-- ## Windows解密错误

## linux子系统没有gcc

## 无法免密码

## known_hosts  authorized_keys  -->