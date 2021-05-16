---
title: "LeetCode 第52场双周赛"
date: 2021-05-16 00:04:36
description: ["LeetCode 第52场双周赛。。 菜死了啊"]
toc: true
author: tabris
# 图片推荐使用图床(腾讯云、七牛云、又拍云等)来做图片的路径.如:http://xxx.com/xxx.jpg
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/16/20210516114449.png
# 如果top值为true,则会是首页推荐文章
top: false
# 如果要对文章设置阅读验证密码的话,就可以在设置password的值,该值必须是用SHA256加密后的密码,防止被他人识破
password:
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛
---

![image-20210516114449615](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/16/20210516114449.png)

GG

# 5742. 将句子排序

简单模拟， 用py实现的，方便点。

```python
class Solution:
    def sortSentence(self, s: str) -> str:
        sl = s.split(' ')
        sl.sort(key=lambda x: x[-1])

        return ' '.join([s[:-1] for s in sl])
```



# 5743. 增长的内存泄露

简单模拟

```cpp
class Solution {
public:
    vector<int> memLeak(int memory1, int memory2) {
        int cnt = 0;
        int i=0;
        for(i=1;memory1 >= i || memory2 >= i; i++) {
            if(memory1 >= memory2) {

            memory1 -= i;
            } else {

            memory2 -= i;
            }

        }
        return {i, memory1, memory2};
    }
};
```



# 5744. 旋转盒子

模拟题，

```cpp
class Solution {
public:
    vector<vector<char>> rotateTheBox(vector<vector<char>>& box) {
        for(auto &raw: box) {
            for(int i=raw.size()-1,p=raw.size()-1; i>=0;i--) {
                if(raw[i] == '*') {
                    p = i-1;
                    continue;
                } else if (raw[i] == '#') {
                    raw[i] = '.';
                    raw[p--] = '#';
                }
            }
        }
        vector<vector<char>> ans;
        int n=box.size(), m = box[0].size();
        for(int j=0;j<m;j++) {
            vector<char> tmp;
            for(int i=n-1;i>=0;i--) {
                tmp.push_back(box[i][j]);
            }
            ans.push_back(tmp);
        }
        return ans;
    }
};
```



# 5212. 向下取整数对和



这题写的比较麻烦，开始写了个分段除法 结果这题居然卡 $O(N \times \sqrt{N})$,

最后改成了类似朴素晒法的过程实现的。

首先将数放入桶中记nums[i]的个数，预处理桶的前缀和， 然后按照晒法的过程途中统计下结果就好了。

这里注意复杂度
$$
\sum_{i=1}^n \frac{n}{i} = n \times log(n)
$$


```cpp
class Solution {
public:
    static const int MOD = 1e9+7;
    // static const int N = 1e5;
    int sumOfFlooredPairs(vector<int>& nums) {
        // sort(nums.begin(), nums.end());
        int N = nums[0];
        // printf("%d --\n", N);
        for(auto a: nums) N = max(N, a);
        // printf("%d --\n", N);
        vector<int> h(N+1);

        int ans = 0;
        int sum = 0;
        for(auto a: nums) h[a]++;
        // for(int i=0;i<=N;i++) printf("%d ",h[i]); puts("");

        for(int i=1;i<=N;i++) {
            h[i] += h[i-1];
        }
        // for(int i=0;i<=N;i++) printf("%d ",h[i]); puts("");

        for(int i=1;i<=N;i++) {
            for(int j=i,k=1; j<=N; j+=i,k++) {
                ans += 1LL*k*(h[i]-h[i-1])%MOD*(h[min(N, j+i-1)]-h[j-1])%MOD;
                // printf("%d >>> %d %d (%d, %d] | %d = %d(k) * %d(h[i]) * %d(h[min(N, j+i-1)]-h[j-1])\n", i, j, k, j-1, min(N, j+i-1), 1LL*k*h[i]%MOD*(h[min(N, j+i-1)]-h[j-1])%MOD, k, h[i], (h[min(N, j+i-1)]-h[j-1]));
                ans %= MOD;
            }
        }
        return ans;
    }
};
```



```cpp
// O(N * Sqrt(N)) 超时的代码
class Solution {
public:
    static const int MOD = 1e9+7;
    // static const int N = 1e5;
    int sumOfFlooredPairs(vector<int>& nums) {
        // sort(nums.begin(), nums.end());
        int N = nums[0];
        // printf("%d --\n", N);
        for(auto a: nums) N = max(N, a);
        // printf("%d --\n", N);
        vector<int> h(N+1);

        int ans = 0;
        int sum = 0;
        for(auto a: nums) h[a]++;
        // for(int i=0;i<=N;i++) printf("%d ",h[i]); puts("");

        for(int i=1;i<=N;i++) {
            ans = (ans + h[i]*h[i]%MOD)%MOD;

            for(int j=1, last; h[i] >0 && j<i; j=last + 1) {
                last = min(i/(i/j), i-1);

                ans += (i/j)*(h[last] - h[j-1])%MOD*h[i]%MOD;
                ans %= MOD;
            }
            h[i] += h[i-1];
            // printf("%d >> %d \n", i, ans);
        }
        // for(int i=0;i<=N;i++) printf("%d ",h[i]); puts("");

        return ans;
    }
};
```
