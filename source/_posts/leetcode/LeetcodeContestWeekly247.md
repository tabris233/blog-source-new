---
title: "LeetCode 第247场周赛"
çdate: 2021-06-27 12:55:17
Updated: 2021-06-27 12:55:17
description: ["LeetCode 第247场周赛。。 再再再再再再再再次翻车"]
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

![image-20210627125617729](https://cdn.jsdelivr.net/gh/tabris233/cdn-assets/PicGo/2021/06/27/20210627125617.png)

>   彻底翻车，，，，，
>
>   数学能力没有了。。。。

# [5797. 两个数对之间的最大乘积差](https://leetcode-cn.com/problems/maximum-product-difference-between-two-pairs/)

因为都是正整数，乘积最大和最小的一定是最大的两个数字的乘积和最小两个数的乘积。

```cpp
class Solution {
public:
    int maxProductDifference(vector<int>& nums) {
        int n = nums.size();
        
        sort(nums.begin(), nums.end());
        
        return nums[n-1]*nums[n-2] - nums[0]*nums[1];
    }
};
```



# [5798. 循环轮转矩阵](https://leetcode-cn.com/problems/cyclically-rotating-a-grid/)

模拟一下获得每个圈的数字然后旋转k次就好。

我这里用搜索搞的， 应该是麻烦了。

```cpp
class Solution {
public:
    queue<int> ceng[55];
    
    int vis[55][55];
    int n,m;
    
    int fx[4] = {0,1,0,-1};
    int fy[4] = {1,0,-1,0};
    void dfs(vector<vector<int>>& grid, int x, int y, int direct, int c, int op) {
        if(x>0 && y>0 && vis[x-1][y] && vis[x][y-1] && vis[x-1][y] == vis[x][y-1]) c++;

        vis[x][y] = c;
        if(op == 0) {
            ceng[c].push(grid[x][y]);
        }
        else {
            grid[x][y] = ceng[c].front(); ceng[c].pop();
        }

        int xx, yy;
        for(int i=direct; i<direct+4; i++) {
            xx = x + fx[i%4];
            yy = y + fy[i%4];

            if(xx < 0 || xx >= n) continue;
            if(yy < 0 || yy >= m) continue;
            if(vis[xx][yy]) continue;

            vis[xx][yy] = 1;
            dfs(grid, xx, yy, i, c, op);
        }
    }
    
    vector<vector<int>> rotateGrid(vector<vector<int>>& grid, int k) {
        n = grid.size(), m = grid[0].size();
        memset(vis, 0, sizeof(vis));
        dfs(grid, 0, 0, 0, 1, 0);
        
        // for(int i=0;i<n;i++) {
        //     for(int j=0;j<n;j++) {
        //         printf("%d ", vis[i][j]);
        //     }
        //     puts("");
        // }
        
        for(int i=1, tmp;i<=50; i++) {
            int l = ceng[i].size();
            if(l == 0) continue;
            // printf("%d <--\n", i);
            int start = k%ceng[i].size();
            for(int j=0; j<start; j++) {
                tmp = ceng[i].front(); ceng[i].pop();
                ceng[i].push(tmp);
            }
        }
        
        memset(vis, 0, sizeof(vis));
        dfs(grid, 0, 0, 0, 1, 1);
        
        return grid;
    }
};
```



# [5799. 最美子字符串的数目](https://leetcode-cn.com/problems/number-of-wonderful-substrings/)

我这里直接无脑统计了， 因为只需要考虑奇偶，用二进制数表示每个字母的出现次数的奇偶性，并记录下前缀异或和即可。

然后暴力FWT计算一下。

>   赛后看了大佬的做法， 又一个小点， 当前n位置的前缀异或和为x的时候，只有与x异或的二进制位上1的个数小于等于1的情况才需要统计。代码就简洁多了。

```cpp
class Solution {
public:
    typedef long long int LL;
    void FWT(LL a[],int n){
        for(int d=1;d<n;d<<=1)
            for(int m=d<<1,i=0;i<n;i+=m)
                for(int j=0;j<d;j++){
                    LL x=a[i+j],y=a[i+j+d];
                    a[i+j]=x+y,a[i+j+d]=x-y;
                }
    }
    void UFWT(LL a[],int n){
        for(int d=1;d<n;d<<=1)
            for(int m=d<<1,i=0;i<n;i+=m)
                for(int j=0;j<d;j++){
                    LL x=a[i+j],y=a[i+j+d];
                    a[i+j]=(x+y)/2,a[i+j+d]=(x-y)/2;
                }
    }
    void solve(LL a[],int n){
        FWT(a,n);
        for(int i=0;i<n;i++) a[i]=a[i]*a[i];
        UFWT(a,n);
    }
    
    LL vis[1<<10];
    
    long long wonderfulSubstrings(string word) {
        int ls = word.size();

        vector<int> cnt(ls+1);

        vis[0] ++;
        for(int i=0;i<ls;i++) {
            cnt[i+1] = cnt[i] ^ (1<<(word[i]-'a'));
            vis[cnt[i+1]] ++;
        }
        
        // FWT
        solve(vis, 1<<10);
        
        LL ans = vis[0] - ls - 1;
        for(int i=0;i<10;i++) {
            ans += vis[1<<i];
        }
        
        return ans/2;
    }
};
```

简洁做法

```cpp
class Solution {
public:
    int cnt[1<<10];
    long long wonderfulSubstrings(string s) {
        cnt[0]++; long long ret=0;
        int cur=0;
        for(char c:s) {
            cur^=(1<<c-'a');
            for(int i=0;i<10;++i) ret+=cnt[cur^(1<<i)];
            ret+=cnt[cur];
            cnt[cur]++;
        }
        return ret;
    }
};
```



# [5800. 统计为蚁群构筑房间的不同顺序](https://leetcode-cn.com/problems/count-ways-to-build-rooms-in-an-ant-colony/)

>   开始还在搞拓扑图，但实际上这玩意就是一个以0为根的树。。。。

蚂蚁能够构建的顺序需要保证父节点在前。

设以节点$x$为根的子树中，他的孩子节点为$to_i$，所有子节点个数为$sz_{x}$；

以节点x为根的子树中，所有可行的排序为$ans_x$;
$$
ans_x = \prod_{to_i} ans_{to_i} \times  \frac{sz_{x} !}{\prod_{to_i} sz_{to_i} !}
$$


```cpp
class Solution {
public:
    typedef long long int LL;

    static const int MOD = 1e9+7;
    static const int N   = 1e5+7;
    
    // C(n, m)
    LL Fac[N],Inv[N];
    LL qmod(LL a,LL b){
        LL res = 1ll;
        for(;b;b>>=1,a=a*a%MOD)
            if(b&1) res=res*a%MOD;
        return res;
    }
    void init(){
        Fac[0] = 1;
        for (LL i = 1; i < N; i++) Fac[i] = (Fac[i-1] * i) % MOD;
        Inv[N-1] = qmod(Fac[N-1], MOD-2);  //Fac[N]^{MOD-2}
        for (LL i = N - 2; i >= 0; i--) Inv[i] = Inv[i+1] * (i + 1) % MOD;
    }
    
    vector<int> g[100005];
    
    LL sz[1000005];
    
    LL dfs(int u) {
        sz[u] = 1;
        LL sumu = 1, sumz;
        LL fz = 1;
        for(auto to: g[u]) {
            sumz = dfs(to);
            
            sz[u] += sz[to];
            
            sumu = sumu*sumz %MOD;
            fz = fz*Inv[sz[to]] %MOD;
        }
        return Fac[sz[u]-1] * sumu %MOD * fz %MOD;
    }

    int waysToBuildRooms(vector<int>& prevRoom) {
        init();
        int n = prevRoom.size();

        for(int i=1; i<n; i++) g[prevRoom[i]].push_back(i);
        
        return dfs(0);
    }
};
```

