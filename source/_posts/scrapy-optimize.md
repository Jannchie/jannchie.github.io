---
title: Scrapy爬虫优化
tags: [爬虫, Scrapy, jieba, 优化, 内存]
categories: 形而上的最佳实践
---

BiliOB 观测者的最近一次更新后，服务器内存占用飞速提高，经常因为内存不足 Kill 掉 Mongodb 数据库以及 Spring boot 的后端服务。

出现这种问题是在爬虫更新后。我在爬虫中引入了结巴分词模块，对弹幕进行分析。

然而 Scrapy 的爬虫管理很有问题，为了引入我需要的分词和词频统计功能，我必须在弹幕爬虫中引入 `jieba.analyze` 这个包，从而在弹幕爬虫中使用。而且在弹幕爬虫的初始化内载入自定义字典。

而实际运行情况是这样的，无论我使用 `scrapy crawl ...` 命令启动哪一个爬虫，都会载入 jieba.analyze，并且初始化每一个爬虫。

而我使用 `memory_profiler` 对内存进行分析后，发现内存的使用情况是这样的：

```bash
Line #    Mem usage    Increment   Line Contents
================================================
31    180.3 MiB    180.3 MiB       @profile
32                                 def __init__(self):
33
34                                     # 读取自定义字典
35    246.7 MiB     66.3 MiB           jieba.load_userdict('.biliob_analyzer/dict.txt')
36
37                                     # 链接mongoDB
38    246.7 MiB      0.0 MiB           self.client = MongoClien(settings['MINGO_HOST'], 27017)
39                                     # 数据库登录需要帐号密码
40    246.7 MiB      0.0 MiB           self.client.admin.authenticate(settings['MINGO_USER'],
41    246.7 MiB      0.0 MiB                                          settings['MONGO_PSW'])
42    246.7 MiB      0.0 MiB           self.db = self.clien['biliob']  # 获得数据库的句柄
43    246.7 MiB      0.0 MiB           self.coll = self.d['author']  # 获得collection的句柄
44    246.7 MiB      0.0 MiB           self.redis_connection = redis.from_url(redis_connect_string)
```

这是把这样一个爬虫加入爬虫列表后，所有爬虫的启动时内存消耗。

此前，一个爬虫大概占据 70MB 左右，引入了结巴分词后，内存消耗直接飙到 180.3MB，上涨 100MB 以上，引入自定义词库之后，再增加 66.3MB，总计消耗 246.7MB。

看上去不多，但是对于我这个非常低端的服务器而言，是非常重的负载了。因为最为糟糕的是，我的服务器有可能会同时跑四五个爬虫，而 java 和 mongodb 也是两个吃内存的怪兽。与之相对，我的服务器只有 2GB 的内存空间。

因此急需优化。但 scrapy 似乎没有内置解决方案。其实有一种很简单的思路，只需要将爬虫分成两个 project，就能不受干扰地引入各自的模块。

经过分解，原先爬虫的内存占用如下。

```bash
Line #    Mem usage    Increment   Line Contents
================================================
29     72.2 MiB     72.2 MiB       @profile
30                                 def __init__(self):
31
32                                     # 链接mongoDB
33     72.3 MiB      0.0 MiB           self.client = MongoClient(settings['MINGO_HOST'], 27017)
34                                     # 数据库登录需要帐号密码
35     72.3 MiB      0.0 MiB           self.client.admin.authenticate(settings['MINGO_USER'],
36     72.3 MiB      0.0 MiB                                          settings['MONGO_PSW'])
37     72.3 MiB      0.0 MiB           self.db = self.client['biliob']  # 获得数据库的句柄
38     72.3 MiB      0.0 MiB           self.coll = self.db['author']  # 获得collection的句柄
39     72.3 MiB      0.0 MiB           self.redis_connection = redis.from_url(redis_connect_string)
```

可以看到，恢复了正常水平。

---

## 可维护

但这样毕竟可维护性是极差的。因为有很多相同的配置需要重复一遍，那么有没有一种方案可以解决这个问题呢？

……
