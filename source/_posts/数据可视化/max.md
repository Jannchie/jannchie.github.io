---
title: 可视化曲线图最大值算法
tags: [算法]
categories: 自由支配的叹服
date: 2019-03-04
---

我希望曲线图的最大值只会增大不会减小，考虑下列算法：

第一种实现是这样的：

```js
// 获得一个从start开始，end结束，步长为step的数组
let range = (start, end, step) => {
  let current = start;
  let result = [];
  while (current < end) {
    result.push(current - start);
    current += step;
  }
  return result;
};
// 检查点的x轴坐标
let xRange = range(0, width - margin.left - margin.right, 200);
// 坐标最大值为所有检查点的最大值
maxVal = d3.max(
  xRange.map(e =>
    yScale.invert(d3.min(pathsDom.map(path => getYFromX(path, e))))
  )
);
```

这么做很土，实际上在SVG中，对于一条path，要通过其x轴坐标找到y轴坐标需要比较庞大的计算量，会影响动画帧率。

---

于是采用下列方式：

``` js
let maxVal =
  yScale.invert(d3.min(posY)) > maxVal
    ? yScale.invert(d3.min(posY))
    : maxVal;

```

只有最新值大于最大值的时候更新最大值，否则不变，这样就能保证最大值不会减小。
