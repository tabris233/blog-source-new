---
title: "LeetCode 第68场双周赛"
date: 2021-12-25 23:58:30
updated: 2021-12-25 23:58:30
description: ["LeetCode 第68场双周赛。。 翻车啊啊啊啊啊"]
toc: true
author: tabris
# 图片推荐使用图床(腾讯云、七牛云、又拍云等)来做图片的路径.如:http://xxx.com/xxx.jpg
img: https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo2021/12/25/20211225235907.png
# 如果top值为true,则会是首页推荐文章
top: false
# 如果要对文章设置阅读验证密码的话,就可以在设置password的值,该值必须是用SHA256加密后的密码,防止被他人识破
password:
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

![](https://fastly.jsdelivr.net/gh/tabris233/cdn-assets/PicGo2021/12/25/20211225235907.png)

GG 感觉

# [5946. 句子中的最多单词数](https://leetcode-cn.com/contest/biweekly-contest-68/problems/maximum-number-of-words-found-in-sentences/)

遍历每个句子，split一下后的数组元素个数最大的那个就是结果了。

```python
class Solution:
    def mostWordsFound(self, sentences: List[str]) -> int:
        ans = 0
        for s in sentences:
            ans = max(ans, len(s.split(' ')))
        return ans
```

# [5947. 从给定原材料中找到所有可以做出的菜](https://leetcode-cn.com/contest/biweekly-contest-68/problems/find-all-possible-recipes-from-given-supplies/)

对每个菜dfs下去就好了，要注意下循环的情况。

```cpp
class Solution {
public:
    unordered_map<string, int> m;
    unordered_map<string, vector<string> > need;
    unordered_map<string, bool> is;

    bool dfs(string s) {
        if (m[s]&1) return true;
        if (is[s]) return false;
        is[s] = true;

        for (auto ss: need[s]) {
            if (m[ss]&1) continue;
            if (m[ss] == 2 && dfs(ss)) continue;
            return false;
        }

        m[s] |= 1;
        return true;
    }


    vector<string> findAllRecipes(vector<string>& recipes, vector<vector<string>>& ingredients, vector<string>& supplies) {
        m.clear();
        need.clear();

        // puts("");

        for (auto s: recipes)  m[s] = 2;
        for (auto s: supplies) m[s] = 1;

        for (int i=0; i<ingredients.size(); i++) {
            need[recipes[i]] = ingredients[i];
        }

        vector<string> ans;
        for (auto s: recipes) {
            is.clear();
            if (dfs(s)) {
                ans.push_back(s);
            }
        }

        return ans;
    }
};
```

# [5948. 判断一个括号字符串是否有效](https://leetcode-cn.com/contest/biweekly-contest-68/problems/check-if-a-parentheses-string-can-be-valid/)

> 这题写的很混乱。甚至担心是水过去的。。

大概就是如果'('为1，')'为-1。 那么合法序列的前缀和一只是>=0的。结果一定是0的。
考虑locked[i]=1不能变，locked[i]=0能变，那就记录两种前缀和的值，一种只把locked[i]=0时当成'(',一种只把locked[i]=0时当成')'。
记录前缀和的最大最小值， 过程中最大值一定是>=0的。结果一定是0在最大最小值中间的。

正反都判断一遍即可。

```cpp
class Solution {
public:
    bool f(string& s, string& locked) {
        int mx = 0, mn = 0;
        for (int i=0; i<s.size(); i++) {
            if (locked[i] == '0') {
                mx ++;
                mn --;
            } else {
                if (s[i] == '(') mx ++, mn ++;
                else mx --, mn --;
            }
            if (mx<0) return false;
        }
        // printf("%s: %d %d\n", s.c_str(), mn, mx);
        return (mn<=0 && 0<=mx);
    }

    bool canBeValid(string s, string locked) {
        if (s.size() & 1) return false;

        string ss = s, lss = locked;
        reverse(ss.begin(), ss.end());
        reverse(lss.begin(), lss.end());
        for (auto &c: ss) {
            if (c == '(') c=')';
            else c='(';
        }
        // puts("");
        // printf("%s %s\n", s.c_str(), ss.c_str());
        // printf("%s %s\n", locked.c_str(), lss.c_str());

        return f(s, locked) && f(ss, lss);
    }
};
```

# [5949. 一个区间内所有数乘积的缩写](https://leetcode-cn.com/contest/biweekly-contest-68/problems/abbreviating-the-product-of-a-range/)

这题分成$3$个部分，
1. 最高的$5$位数字

    每次保留前12位数字，因为仅保留5位数字可能会少了后面的进位，影响结果。

2. 除$0$外最低的$5$位数字

    正常情况下只要一直对$100000$取模就好了，但是这里要的是除$0$外的末尾$5$个数字。
    但这里可能有结尾是$0$的情况，多保存几位，保留最后12位数字这样去掉末尾0的时候也不会影响结果。

3. 结尾0的个数

    先考虑结尾0的个数，考虑每个因子因式分解后2和5的个数，取最小值即可。

⚠️注意：计算过程中先判断非结尾0的部分是不是大于10位，是的话再处理上述的1和2就好。

```cpp
class Solution {
public:
    int f(int n, int x) {
        int sum = 0;
        while (n%x == 0) {
            n/=x;
            sum++;
        }
        return sum;
    }

    string abbreviateProduct(int left, int right) {
        long long int zero = 0, five = 0, two = 0;
        long long int p = 1, s = 1;
        bool flag=false;
        for (long long int i = left; i<=right; i++) {
            five += f(i, 5);
            two  += f(i, 2);

            p*=i; s*=i;
            while(p%10 == 0) p/=10;
            while(s%10 == 0) s/=10;

            if (!flag && p >= 10000000000LL) {
                flag = true;
            }
            if (flag) {
                while(p>=1000000000000LL) p/=10;
                s %= 1000000000000LL;
            }
        }

        string ans = "";

        if (flag) {
            printf("%lld %lld\n", p, s);
            s %= 100000LL;
            string ss = to_string(s+100000LL);
            ss = ss.substr(1, 5);
            while(p>=100000LL) p/=10;
            ans += to_string(p) + "..." + ss;
        } else {
            ans += to_string(p);
        }

        zero = min(two, five);
        ans += "e" + to_string(zero);
        return ans;
    }
};
```