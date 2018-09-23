[TOC]
## 题目
* 题目：distinct-subsequences
* 题目类型：动态规划

> Given a string S and a string T, count the number of distinct subsequences of T in S.

> A subsequence of a string is a new string which is formed from the original string by deleting some (can be none) of the characters without disturbing the relative positions of the remaining characters. (ie,"ACE"is a subsequence of"ABCDE"while"AEC"is not).

> Here is an example:
S ="rabbbit", T ="rabbit"

> Return3.

题意：给定两个字符串S和T，计算字符序列T在S中出现的次数。

## 题解
这道题只要保证T中字母在S中出现的顺序不变，那么就是一个子序列，比如上边的例子S="rabbbit",T="rabbit"，结果是3，由S中的"bbb"有3种"bb"的组合情况得到。这道题得用动态规划做，因为是要求数量，用dfs递归然后再判断的话肯定超时。
设置dp[i][j]表示T的前i个字符和S的前j个字符的结果数，判断i处T的字符和j处的S的字符是否匹配：
* 不匹配的话dp[i][j]=dp[i][j-1]，意思是，当前T的字符和S的字符不相同，那么S的[0,j]的序列包含T的[0,i]的序列的次数和S的[0,j-1]的序列的包含情况是一样的
* 匹配的话dp[i][j]=dp[i][j-1]+dp[i-1][j-1]，这里dp[i][j-1]的意思是，T的[0,i]序列和S的[0,j-1]序列匹配的结果数，S的j处字符先不考虑；dp[i-1][j-1]的意思是，S的j处的字符考虑，即这里的情况是T的i处的字符一定和S的j处的字符匹配，返回这时候的结果数。

初始化的时候，设置dp[0][j]全为1，表示长度为0的T的序列与S的任何序列都匹配一次。
``` java
public int numDistinct(String S, String T) {

        if (S==null||S.length()==0||T==null||T.length()==0)
            return 0;

        int[][] dp=new int[T.length()+1][S.length()+1];//T的前i个和S的前j的组合数
        for (int i=0;i<S.length();i++)//T为空串时，所以的S位置都可以匹配
            dp[0][i]=1;
        for (int i=1;i<=T.length();i++){
            for (int j=1;j<=S.length();j++){
                if (T.charAt(i-1)!=S.charAt(j-1))//如果当前字符不匹配，那么为当前字符和S的前面子串的匹配数
                    dp[i][j]=dp[i][j-1];
                else dp[i][j]=dp[i][j-1]+dp[i-1][j-1];//当前字符和S的字符不匹配以及匹配的情况的合计
            }
        }
        return dp[T.length()][S.length()];
    }
```