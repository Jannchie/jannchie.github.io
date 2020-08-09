---
title: MongoDB大小写不敏感查询
tags: [数据库, MongoDB]
categories: 自由支配的叹服
date: 2019-03-11
---

Mysql中，查询对于大小写是不敏感的，比如查询`ABc`会同时查询出`abc`,`abC`的值。

然而在MongoDB中，查询对于大小写是敏感的。

对于小规模的数据，我们可以使用`$regex`来解决这一问题：

``` javascript
db.coll.find(
  {
    keyword:{
      $regex:'xxxxx',
      $options:'i'
    }
  })
```

通过将`$option`设为`i`可以进行大小写不敏感的查询。

然而对于非常大的collection，这一种方法就不合适了。

查询[MongoDB官方文档](https://docs.mongodb.com/manual/reference/operator/query/regex/)我们发现：

> Case insensitive regular expression queries generally cannot use indexes effectively. The $regex implementation is not collation-aware and is unable to utilize case-insensitive indexes.

也就是说，这波操作并不能走索引，在数据量比较大的情况下，会有严重的性能问题。

我采用的解决方式是，在保存时，全部进行一次转换，按照小写进行保存，查询时也将用户的输入全部转换为小写。
