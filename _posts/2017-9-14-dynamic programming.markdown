---
layout:     post
title:      "经典动态规划问题"
subtitle:   " \"dynamic programming\""
author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
tags:
    - 算法
---

> 下滑这里查看更多内容

动态规划把多阶段过程转化为一系列单阶段问题，然后求解重叠子问题，找到最优子结构。就可以得到最优的答案。

特点：具有“最优子结构”、“子问题重叠”、“边界”和“子问题独立”  
具有这四个特点的问题就可以使用动态规划来解决。

### 经典问题1：找零钱  
题目介绍到处都有，不重复了。目标钱数为n，共m种零钱面值。  
可以分解问题包含第m种面值或不包含第m种面值  
计数总和为`co(a, m - 1, n) + co(a, m, n - a[m-1])`  
第一项是没用第m种零钱，第二项是用了第m种零钱。  
具体代码如下  
```
public static int co(int[] a, int m, int n) {
	if(n == 0) return 1;
	if(n < 0 || m==0) return 0;
	return co(a, m - 1, n) + co(a, m, n - a[m-1]);
}
```

如果需要将每种方案记录下来，代码如下  
```
import java.util.ArrayList;
import java.util.Arrays;
import java.util.HashSet;
 
public class Demo {

    public static void main(String[] args) {
        int[] arr = {1,2,3,4,5};
        HashSet<ArrayList<Integer>> fun = fun(arr, 8);
        for (ArrayList<Integer> arrayList : fun) {
            System.out.println(arrayList.toString());
        }
    }
 
    private static HashSet<ArrayList<Integer>> fun(int[] arr, int n) {
        HashSet<ArrayList<Integer>> list = new HashSet<>();
        ArrayList<Integer> subList = new ArrayList<Integer>();
        Arrays.sort(arr);
        fun1(list, subList, arr, 0, n);
         
        return list;
    }
 
    private static void fun1(HashSet<ArrayList<Integer>> list,
            ArrayList<Integer> subList, int[] arr, int i, int n) {
        if (n == 0) {
            list.add(subList);
            return;
        }
        if(n<0)
            return ;
             
        for (int j = i; j < arr.length; j++) {
            ArrayList<Integer> newSub = new ArrayList<Integer>(subList);
            newSub.add(arr[j]);
            fun1(list, newSub, arr,j, n-arr[j]);
        }
    }
}
```
当要求零钱数目最少时，就转化为贪心算法  
```
public static int[] greed(int m[], int n) {
    int k = m.length;

    int[] nums = new int[k];
    for(int i = 0; i < k; i++) {
            nums[i] = n/m[i];
            n = n % m[i];
    }
    return nums;
}
 ```
nums存储各种零钱的数目。  
**注意**：m要降序排列





### 经典问题2：01背包问题  
w[i]: 第i个物体的重量；  
p[i]: 第i个物体的价值；  
c[i][m]: 前i个物体放入容量为m的背包的最大价值；  
c[i-1][m]: 前i-1个物体放入容量为m的背包的最大价值；  
c[i-1][m-w[i]]: 前i-1个物体放入容量为m-w[i]的背包的最大价值；  
可得关系式：`c[i][m] = Math.max(c[i-1][m-w[i]]+p[i], c[i-1][m]);`  
由关系式可得最优子结构。
打表代码如下：  
```
package test;

public class BackPack {
    public static void main(String[] args) {
        int m = 12;
        int n = 8;
        int w[] = {2,1,3,2,4,5,3,1};
        int p[] = {13,10,24,15,28,33,20,8};
        int c[][] = BackPack_Solution(m, n, w, p);
        for (int i = 1; i <=n; i++) {
            for (int j = 1; j <=m; j++) {
                System.out.print(c[i][j]+"\t");
                if(j==m){
                    System.out.println();
                }
            }
        }
        System.out.print(c[n][m]);
    }

    /**
     * @param m 表示背包的最大容量
     * @param n 表示商品个数
     * @param w 表示商品重量数组
     * @param p 表示商品价值数组
     */
    public static int[][] BackPack_Solution(int m, int n, int[] w, int[] p) {
        //c[i][m]表示前i件物品恰放入一个重量为m的背包可以获得的最大价值
        int c[][] = new int[n + 1][m + 1];

        for (int i = 1; i < n + 1; i++) {
            for (int j = 1; j < m + 1; j++) {
                //当物品为i件重量为j时，如果第i件的重量(w[i-1])小于重量j时，c[i][j]为下列两种情况之一：
                //(1)物品i不放入背包中，所以c[i][j]为c[i-1][j]的值
                //(2)物品i放入背包中，则背包剩余重量为j-w[i-1],所以c[i][j]为c[i-1][j-w[i-1]]的值加上当前物品i的价值
                if (w[i - 1] <= j) {
                    c[i][j] = Math.max(c[i - 1][j], c[i - 1][j - w[i - 1]] + p[i - 1]);
                } else
                    c[i][j] = c[i - 1][j];
            }
        }
        return c;
    }
}

```
### 经典问题3：最长公共子序列  
这里的子序列是指从该字符串中去掉任意多个字符后剩下的字符在不改变顺序的情况下组成的新子字符串。  
例如A = "abcdef"和B = "adefcb" 的最长公共子序列就是adef。  
最优子结构为:  
`if(s1.charAt(i) == s2.charAt(j)) c[i,j] = c[i-1][j-1] + 1;`  
`if(s1.charAt(i) != s2.charAt(j)) c[i,j] = Math.max(c[i][j-1], c[i-1][j]);`  
先做备忘录，利用备忘录可以找到最长公共子序列字符，代码如下：  
```
package test;

public class t {

    public static void main(String[] args) {
        String str1 = "abcdef";
        String str2 = "adefcb";
        // 做备忘录
        int[][] dp = LCS(str1, str2);
        // 打印矩阵
        for (int i = 0; i <= str1.length(); i++) {
            for (int j = 0; j <= str2.length(); j++) {
                System.out.print(dp[i][j] + "   ");
            }
            System.out.println();
        }
        // 输出LCS
        print(dp, str1, str2, str1.length(), str2.length());
    }

    public static int[][] LCS(String str1, String str2) {
        int[][] dp = new int[str1.length() + 1][str2.length() + 1];
        for (int i = 1; i <= str1.length(); i++) {
            for (int j = 1; j <= str2.length(); j++) {
                if (str1.charAt(i - 1) == str2.charAt(j - 1)) {
                    dp[i][j] = dp[i - 1][j - 1] + 1;
                } else {
                    dp[i][j] = (dp[i - 1][j] >= dp[i][j - 1] ? dp[i - 1][j] : dp[i][j - 1]);
                }
            }
        }
        return dp;
    }

    // 根据矩阵输出LCS
    public static void print(int[][] opt, String s1, String s2, int i, int j) {
        if (i == 0 || j == 0) {
            return;
        }
        if (s1.charAt(i - 1) == s2.charAt(j - 1)) {
            print(opt, s1, s2, i - 1, j - 1);
            System.out.print(s1.charAt(i - 1));
        } else if (opt[i - 1][j] >= opt[i][j]) {
            print(opt, s1, s2, i - 1, j);
        } else {
            print(opt, s1, s2, i, j - 1);
        }
    }
}
```




参考：
[金矿模型](http://www.jianshu.com)