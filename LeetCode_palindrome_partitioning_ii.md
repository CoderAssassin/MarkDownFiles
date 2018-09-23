[TOC]
## 题目
* 题目：palindrome-partitioning-ii
* 题目类型：动态规划

> Given a string s, partition s such that every substring of the partition is a  palindrome.
> Return the minimum cuts needed for a palindrome partitioning of s.
> For example, given s ="aab",
> Return 1 since the palindrome partitioning["aa","b"]could be produced using 1 cut.

题意：给定一个字符串s，对s进行划分，使得每个划分都是回文，要求返回最少的切割次数。

## 题解
这道题，求最小的划分次数，用动态规划。根据动态规划题的一般的做法，先推导出状态转移方程，用递归尝试解，发现爆栈，然后用迭代的方式解。这里关键的地方是如何推导出状态转移方程，也是动态规划的核心问题。
看本题的样例，s="aab"，那么一般的，我们从左到右遍历，先切一刀，然后再对剩余的子部分切第二刀，再继续下去。切一刀有两种组合["a","ab"]和["aa","b"]，然后再分别判断每种组合的子部分是不是回文，对"ab"继续切，那么第一种组合也就变成了["a","a","b"]，全都是回文了，总的来说有两种切法["a","a","b"]和["aa","b"]，分别是切两刀和一刀。看上边的分析过程，其实是个递归回溯的过程，我们一般写一个dfs()函数来递归回溯，但是一般这样的题用递归回溯很容易超时或者爆栈，因此就要用迭代的方式来解。**动态规划的迭代核心思路就是后边的计算都是用到前面的计算结果的**。
这道题的状态转移方程是：用dp[i]表示0-i的子串的最小的切割数，那么在遍历的时候，若当前位置i，从当前位置往前遍历到j
* 若[j,i]的子串是回文串，那么dp[i]=d[j-1]+1，即i处的最小切割数等于dp[j-1]处的最小切割数加上j-1与j之间的那一刀；
* 若[j,i]的子串不是回文串，那么dp[i]=dp[j-1]+i-j+1，意思是，[j,i]的每两个字符之间都要切一刀。

最后是初始化的问题，初始化判断[0,i]是否回文，是的话最小切割为0，否则为字符数，即最大切割。
``` java
	public int minCut(String s) {

        if (s.equals("")||s==null||s.length()==1)
            return 0;

        int[] dp=new int[s.length()];//记录0-i的子字符串的最小切割数

        for (int i=1;i<s.length();i++){//i=0的时候肯定是0，所以下标从1开始
            dp[i]=isPalindrome(s.substring(0,i+1))?0:i;//初始化
            if (dp[i]==0)//表明[0,i]是回文串，直接跳过
                continue;
            for (int j=1;j<=i;j++){//遍历前面的子串，更新dp
                if (isPalindrome(s.substring(j,i+1)))
                    dp[i]=Math.min(dp[i],dp[j-1]+1);
                else dp[i]=Math.min(dp[i],dp[j-1]+i-j+1);
            }
        }
        return dp[s.length()-1];
    }

    private boolean isPalindrome(String s){
        int start=0,end=s.length()-1;
        while (start<end){
            if (s.charAt(start)!=s.charAt(end))
                return false;
            start++;
            end--;
        }
        return true;
    }
```
