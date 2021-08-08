---
title: "LeetCode 第246场周赛"
date: 2021-06-27 15:26:33
updated: 2021-06-27 15:26:33
description: ["LeetCode 第246场周赛。。 虚拟"]
toc: true
author: tabris
img: https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/05/30/20210530121215.png
katex: true
summary:
categories:
    - leetcode
tags:
    - 周赛
    - 双周赛

---

![image-20210627151634551](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/06/27/20210627151634.png)

>   这场是虚拟的，还是在床上躺着优哉游哉的打的，，成绩居然还可以？
>
>   这是巧合还是，，，

# [1903. 字符串中的最大奇数](https://leetcode-cn.com/contest/weekly-contest-246/problems/largest-odd-number-in-string/)

找到最后一个是奇数的位置作为结尾即可

```go
func largestOddNumber(num string) string {
    ls := len(num)

    end := -1
    for i:=ls-1; i>=0; i-- {
        if (int(num[i]) - int('0'))&1 == 1 {
            end = i
            break
        }
    }

    return num[:end+1]
}
```



# [1904. 你完成的完整对局数](https://leetcode-cn.com/contest/weekly-contest-246/problems/the-number-of-full-rounds-you-have-played/)

简单算下就好，

都转成分钟，startTime向上取整，finishTime向下取整。 然后算差即可

注意两个点：

1.  跨天的情况
2.  取整后finishTime小于startTime的情况

```cpp
class Solution {
public:
    int toIntTime(string s) {
        int ans = (s[0]-'0')*10+(s[1]-'0');
        ans = ans * 60 + (s[3]-'0')*10+(s[4]-'0');

        return ans;
    }

    int numberOfRounds(string startTime, string finishTime) {
        int s = toIntTime(startTime);
        int e = toIntTime(finishTime);

        printf("%d %d  <<-- \n", s, e);
        if(e < s) e += 24*60;

        s = (s+15-1)/15;
        e = e/15;

        printf("%d %d\n", s, e);

        return (e-s)>0?(e-s):0;
    }
};
```





# [1905. 统计子岛屿](https://leetcode-cn.com/contest/weekly-contest-246/problems/count-sub-islands/)

搜索一下grid2即可，中间记录下结果

```cpp
class Solution {
public:
    int n, m;

    int fx[4] = {0,0,1,-1};
    int fy[4] = {1,-1,0,0};

    int vis[505][505];
    int dfs(vector<vector<int>>& grid1, vector<vector<int>>& grid2, int x, int y) {
        vis[x][y] = 1;

        int ans = 1;
        int xx, yy;
        for(int i=0;i<4;i++) {
            xx = x + fx[i];
            yy = y + fy[i];

            if(0 > xx || xx >= n) continue;
            if(0 > yy || yy >= m) continue;
            if(vis[xx][yy] || grid2[xx][yy] == 0) continue;

            ans &= grid1[xx][yy];
            ans &= dfs(grid1, grid2, xx, yy);
        }

        return ans;
    }

    int countSubIslands(vector<vector<int>>& grid1, vector<vector<int>>& grid2) {
        n = grid1.size(), m = grid1[0].size();

        int ans = 0;
        for(int i=0;i<n;i++) for(int j=0;j<m;j++) {
            if(grid2[i][j] == 1 && grid1[i][j] == 1 && vis[i][j] == 0) {
                ans += dfs(grid1, grid2, i, j);
            }
        }

        return ans;
    }
};
```



# [1906. 查询差绝对值的最小值](https://leetcode-cn.com/contest/weekly-contest-246/problems/minimum-absolute-difference-queries/)

因为值域只有100，所以统计100个前缀和方便快速计算出在任意一个区间这个是否有过即可。

总复杂度是$O(n\times 100 + q \times 100)$  也就是 $O(n + q)$，不过常熟比较大。

```cpp
class Solution {
public:
    int n;
    int sum[101][100005];

    vector<int> minDifference(vector<int>& nums, vector<vector<int>>& queries) {
        n = nums.size();

        for(int i=0; i<n; i++) {
            int a = nums[i];
            sum[a][i+1] ++;

            for(int j=1;j<=100;j++) sum[j][i+1] += sum[j][i];
        }

        vector<int> ans;
        for(auto q: queries) {
            int l = q[0]+1, r = q[1]+1;

            int pre = -1;
            int ret = -1;
            for(int i=1; i<=100; i++) {
                if(sum[i][r]-sum[i][l-1] > 0) {
                    if(pre != -1) {
                        if(ret == -1) ret = 111;
                        ret = min(ret, i-pre);
                    }
                    pre = i;
                }
            }
            ans.push_back(ret);
        }

        return ans;
    }
};
```
