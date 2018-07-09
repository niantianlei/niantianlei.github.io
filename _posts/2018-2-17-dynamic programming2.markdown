---
layout:     post
title:      "动态规划二"
subtitle:   " \"dynamic programming II\""

author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
header-mask: 0.4
catalog:    true
tags:
    - 算法
---
> 下滑这里查看更多内容

很久之前看了[背包九讲](https://github.com/tianyicui/pack)，对动态规划问题的理解有所加深（最后附有背包九讲的Java代码）。前几天又看了一些动态规划的其他经典题目，这里做个总结。  

解决动态规划问题一般分为两步：  
1.确定问题可以用动态规划思想解决，之前写过一个[入门博客](http://niantianlei.com/2017/09/14/dynamic-programming/)里面有所涉及，要想真正掌握我觉得还是需要多看、多写、多思考；  
2.找规律，总结状态转移方程，进而写出伪代码。  

下面看具体例子吧

## 问题1——找零钱
之前的博客写过这个问题，但是是针对找零钱的方法数量。  
现在讨论最少货币找出零钱。  
#### 每种面值无限使用
每种问题无限使用的条件下，其实与多重背包问题等价，面值相当于物品的重量，区别就是需要重量正好等于背包容量。我们使用背包九讲中的解法，改变初始值，即可解决。  

转移方程为dp[i][j] = min{dp[i-1][j], dp[i][j-arr[i]]+1}
```
public static int coinChange(int[] coins, int amount) {
    int max = Integer.MAX_VALUE;
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, max);
    dp[0] = 0;
    for(int i = 0; i < coins.length; i++) {
        for(int j = coins[i]; j < amount + 1; j++) {
            if(dp[j-coins[i]] != max) {
                dp[j] = Math.min(dp[j], dp[j-coins[i]] + 1);
            }
        }
    }
    return dp[amount] == max ? -1 : dp[amount];
}
```
#### 每个面值只使用一次 
同样地，此问题与01背包等价。参考背包问题，只需要将循环的顺序改变即可，  
这是因为，此时的转移方程：dp[i][j] = min{dp[i-1][j], dp[i-1][j-arr[i]]+1}  
```
for(int i = 0; i < coins.length; i++) {
    for(int j = amount; j >= coins[i]; j--) {
        if(dp[j-coins[i]] != max) {
            dp[j] = Math.min(dp[j], dp[j-coins[i]] + 1);
        }
    }
}
``` 
最后附上动态规划解决找零钱方法数量问题，此方法是对之前博客方法的优化  
```
int[] dp = new int[amount + 1];
	for(int i = 0; coins[0] * i <= amount; i++) {
		dp[coins[0] * i] = 1;
	}
	for(int i = 1; i < coins.length; i++) {
		for(int j = coin[i]; j <= amount; j++) {
			dp[j] += dp[j - coins[i]];
		}
	}
return dp[amount];
```
## 问题2——最长公共子串
用一个二维数组c[][]，记录以str1(i-1)和str2(j-1)为公共子串最后一个字符的最长公共子串长度。  
与最长公共子序列不同的是，这里数组最右下方的元素不是答案，因为此时的动态规划表的意义产生了变化。  
因此，转移方程：  
str1.charAt(i-1)等于str2.charAt(j-1)，c[i][j]=c[i-1][j-1]+1，  
否则，c[i][j]=0.  
```
int len1 = str1.length();  
int len2 = str2.length();  
int result = 0;     //记录最长公共子串长度  
int c[][] = new int[len1+1][len2+1]; 
for (int i = 0; i <= len1; i++) {  
    for( int j = 0; j <= len2; j++) {  
        if(i == 0 || j == 0) {  
            c[i][j] = 0;  
        } else if (str1.charAt(i-1) == str2.charAt(j-1)) {  
            c[i][j] = c[i-1][j-1] + 1;  
            result = Math.max(c[i][j], result);  
        } else {  
            c[i][j] = 0;
        }  
    }  
}  
```
要找这个子串就非常容易了，找到最长时公共子串的尾字符，然后向前数result长度即为最长的公共子串。  
```
int end = 0;
for(int i = 1; i <= len1; i++) {
	for(int j = 1; j <= len2; j++) {
		if(result == c[i][j]) {
			end = i;
		}
	}
}
String s = str1.substring(end - result, end);
```
## 问题3——最长递增子序列
用dp[i]表示以dp[i]结尾的最长递增子序列的长度，  
转移方程为dp[i] = Math.max(dp[i], max{dp[j], i<j && arr[j]<arr[i]} + 1)  
```
int[] dp = new int[arr.length];
for(int i = 0; i < arr.length; i++) {
	dp[i] = 1; 
	for(int j = 0; j < i; j++) {
		if(arr[i] > arr[j]) {
			dp[i] = Math.max(dp[i], dp[j] + 1);
		}
	}
}
```
最长递增子序列的寻找，先找到最长递增子序列的尾字符所在位置index，然后从此位置向前遍历，如果dp[i]正好是dp[index]-1说明子序列用上了该字符，记录下来。记录下LIS长度的字符后，即完成。  
```
int index = 0;      //最大的位置，从
int len = 0;        //最长递增子序列长度
for(int i = 0; i < dp.length; i++) {
	if(dp[i] > len) {
		len = dp[i];
		index = i;
	}
}
int[] lis = new int[len];       //最长子序列
lis[--len] = arr[index];
for(int i = index; i >= 0; i--) {
	if(arr[i] < arr[index] && dp[i] == dp[index] - 1) {
		lis[--len] = arr[i];
		index = i;
	}
}
```

##### 背包九讲的Java解：   
```
//01背包，解一    复杂度O(m*N)
public static int bagg01(int[] c, int[] w, int m) {
	int N = c.length;
	int[][] res = new int [N+1][m+1];
	for(int i = 1; i < N+1; i++) {
		for(int j = 1; j < m+1; j++) {
			if(j >= c[i-1]) {
				res[i][j] = Math.max(res[i-1][j], res[i-1][j-c[i-1]] + w[i-1]);
			} else {
				res[i][j] = res[i-1][j];
			}
		}
	}
	return res[N][m];
}
//01背包，解二。空间复杂度O(m)
public static int bag01(int[] c, int[] w, int m) {
	int N = c.length;
	int[] res = new int[m+1];
	for(int i = 1; i <= N; i++) {
		for(int j = m; j >= c[i-1]; j--) {
			res[j] = Math.max(res[j], res[j-c[i-1]] + w[i-1]);
		}
	}
	return res[m];
}
//完全背包的两个解,空间复杂度分别为Nm,m
public static int bag1(int[] c, int[] w, int m) {
	int N = c.length;
	int[][] res = new int[N+1][m+1];
	for(int i = 0; i <= m; i++) {
		res[0][i] = 0;
	}
	for(int i = 1; i <= N; i++) {
		for(int j = 1; j <= m; j++) {
			if(j >= c[i-1]) {
				res[i][j] = Math.max(res[i-1][j], res[i][j-c[i-1]] + w[i-1]);
			} else {
				res[i][j] = res[i-1][j];
			}
		}
	}
	return res[N][m];
}
public static int bag2(int[] c, int[] w, int m) {
	int N = c.length;
	int[] res = new int[m+1];
	for(int i = 0; i <= m; i++) {
		res[i] = 0;
	}
	for(int i = 1; i <= N; i++) {
		for(int j = c[i-1]; j <= m; j++) {
			res[j] = Math.max(res[j], res[j-c[i-1]] + w[i-1]);
		}
	}
	return res[m];
}

//多重背包问题
public static int bagmul(int[] c, int[] w, int[] nums, int m) {
	int[] res = new int[m+1];
	int N = c.length;
	for(int i = 1; i <= N; i++) {
		if(c[i-1] * nums[i-1] >= m) {
			//完全背包问题
			for(int j = c[i-1]; j <= m; j++) {
				res[j] = Math.max(res[j], res[j-c[i-1]] + w[i-1]);
			}
		} else {
			int k = 1;
			while(k < nums[i-1]) {
				//01背包问题
				for(int j = m; j >= k*c[i-1]; j--) {
					res[j] = Math.max(res[j], res[j-k*c[i-1]] + k*w[i-1]);
				}
				nums[i-1] -= k;
				k = 2*k;
			}
			//01背包问题
			for(int j = m; j >= c[i-1] * nums[i-1]; j--) {
				res[j] = Math.max(res[j], res[j-nums[i-1]*c[i-1]] + nums[i-1] * w[i-1]);
			}
		}
	}
	return res[m];
}

//记录最优解
public static int[][] bag99(int[] c, int[] w, int m) {
	int N = c.length;
	int[] res = new int[m+1];
	int[][] G = new int[N+1][m+1];
	for(int i = 0; i <= m; i++) {
		res[i] = 0;
	}
	for(int i = 1; i <= N; i++) {
		for(int j = m; j >= c[i-1]; j--) {
			if(res[j] < res[j-c[i-1]] + w[i-1]) {
				res[j] = res[j-c[i-1]] + w[i-1];
				G[i][j] = 1;
			}  
		}
	}
	return G;
}
```