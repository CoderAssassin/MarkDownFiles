[TOC]
## 题目
* 题目：edit-distance
* 题目类型：动态规划

> Given two words word1 and word2, find the minimum number of steps required to convert word1 to word2. (each operation is counted as 1 step.)

> You have the following 3 operations permitted on a word:

> a) Insert a character
b) Delete a character
c) Replace a character

题意：给定两个单词word1和word2，求编辑距离

## 题解
动态规划题，算两个单词的编辑距离，设定一个二维数组dp[i][j]表示word1的前i个字符和word2的前j个字符的最小编辑距离。循环遍历两个单词字符串，当当前的字符相等时，dp[i][j]=dp[i-1][j-1]，意思是当前最小编辑距离和子串编辑距离相等；当当前的字符不相等的时候，dp[i][j]=Math.min(Math.min(dp[i-1][j],dp[i][j-1]),dp[i-1][j-1])+1，意思是当前的情况分成两种，一种是添加或者删除一个字符，一种是替换一个字符，这两种情况对编辑距离的影响是加1。
初始值设定为当word1和word2分别为空串时的编辑距离。
``` java
public int minDistance(String word1, String word2) {
        if (word1==null&&word2==null)
            return 0;
        if (word1==null||word1.equals(""))
            return word2.length();
        if (word2==null||word2.equals(""))
            return word1.length();

        int[][] dp=new int[word1.length()+1][word2.length()+1];
        dp[0][0]=0;//都是空串

        for (int i=1;i<=word1.length();i++)//当word2为空串时
            dp[i][0]=i;
        for (int i=1;i<=word2.length();i++)//当word1为空串时
            dp[0][i]=i;

        for (int i=1;i<=word1.length();i++){
            for (int j=1;j<=word2.length();j++){
                if (word1.charAt(i-1)==word2.charAt(j-1))//当当前的字符相等的时候，那么该字符匹配上，结果为前面字符串的最小匹配
                    dp[i][j]=dp[i-1][j-1];
                else
//                    dp[i-1][j]和dp[i][j-1]和dp[i-1][j-1]表示最后一个字符不匹配的时候，或者删除或者添加或者替换，这样最后要加个1
                    dp[i][j]=Math.min(Math.min(dp[i-1][j],dp[i][j-1]),dp[i-1][j-1])+1;
            }
        }
        return dp[word1.length()][word2.length()];
    }
```

