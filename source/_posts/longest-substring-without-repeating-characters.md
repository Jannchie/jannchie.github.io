---
title: 滑动窗口算法-无重复字符的最长子串
tags: [算法]
categories: 自由支配的叹服
---

无重复字符的最长子串是一个非常简单的问题。实现起来不难，但如果想要高效率地实现这个功能，其中的算法并没有那么直观。

下面是一个使用了滑动窗口算法的神奇的实现：

```python
class Solution:
    def lengthOfLongestSubstring(self, s):
        """
        :type s: str
        :rtype: int
        """
        st = {}
        i, ans = 0, 0
        for j in range(len(s)):
            if s[j] in st:
                i = max(st[s[j]], i)
            ans = max(ans, j - i + 1)
            st[s[j]] = j + 1
        return ans
```
