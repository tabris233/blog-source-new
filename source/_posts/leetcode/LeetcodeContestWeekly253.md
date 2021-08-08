---
title: "LeetCode 第253场周赛"
date: 2021-08-08 11:31:33
updated: 2021-08-08 11:31:33
description: ["LeetCode 第253场周赛。。 翻车翻到姥姥家啊啊啊啊啊。"]
toc: true
author: tabris
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/08/08/20210808112249.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

![image-20210808112249489](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/08/08/20210808112249.png)

# [5838. 检查字符串是否为数组前缀](https://leetcode-cn.com/contest/weekly-contest-253/problems/check-if-string-is-a-prefix-of-array/)

md 开始读错题了。。

其实就是把words的前缀拼一起 看有没有等于s的就好

```cpp
class Solution {
public:
    bool isPrefixString(string s, vector<string>& words) {
        string ss;
        for(auto w: words) {
            ss += w;
            if(ss == s) return true;
        }

        return false;
    }
};
```



# [5839. 移除石子使总数最小](https://leetcode-cn.com/contest/weekly-contest-253/problems/remove-stones-to-minimize-the-total/)

这题很简单， 维护一个队头最大的优先队列。

每次取最大的元素$x$，也就是队头，减去$\lfloor  \frac{x}{2} \rfloor$就好了。

```cpp
class Solution {
public:
    int minStoneSum(vector<int>& piles, int k) {
        priority_queue<int> q;

        int sum = 0;
        for(auto p: piles) {
           q.push(p);
            sum += p;
        }

        for(int i=0; i<k && !q.empty(); i++) {
            int mx = q.top(); q.pop();
            sum -= mx/2;
            mx -= mx/2;

            q.push(mx);
        }

        return sum;
    }
};
```



# [5840. 使字符串平衡的最小交换次数](https://leetcode-cn.com/contest/weekly-contest-253/problems/minimum-number-of-swaps-to-make-the-string-balanced/)

维护下 括号是否匹配， 如果不匹配就把最后一个 `'['` 换过来就好。

```cpp
class Solution {
public:
    int minSwaps(string s) {
        int ls = s.length();
        int ln = ls - 1;

        int sum = 0, ans = 0;
        for(int i=0; i<ln; i++) {
            if (s[i] == ']') {
                sum -= 1;
            } else {
                sum += 1;
            }

            if(sum < 0) {
                while(i<ln && s[ln] == ']') ln--;
                swap(s[i], s[ln]);
                sum += 2;

                ans ++;
            }
        }

        return ans;
    }
};
```

# [5841. 找出到每个位置为止最长的有效障碍赛跑路线](https://leetcode-cn.com/contest/weekly-contest-253/problems/find-the-longest-valid-obstacle-course-at-each-position/)

md, lis 写不明白了。。。

这题就是求每个元素在`最长不下降子序列中的序号`。

```cpp
func longestObstacleCourseAtEachPosition(obstacles []int) []int {
    // fmt.Println("---------------------------")
    ans := make([]int, 0)

    // lis
    lis := make([]int, 0)
    for _, v := range obstacles {
        p := 0

        if len(lis) == 0 || lis[len(lis)-1] <= v {
            lis = append(lis, v)

            p = len(lis)
        } else {
            pos := func() int {
                l, r, mid, ans := 0, len(lis)-1, -1, 0
                for l<=r {
                    mid = (l+r) >> 1
                    if lis[mid] > v {
                        r, ans = mid-1, mid
                    } else {
                        l = mid+1
                    }
                }
                return ans
            }()

            lis[pos] = v

            p = pos + 1
        }

        // fmt.Println(lis)

        ans = append(ans, p)
    }


    return ans
}
```
