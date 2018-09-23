[TOC]
## 题目
* 题目：subsets
* 题目类型：动态规划

> Given a set of distinct integers, S, return all possible subsets.

> Note:

> Elements in a subset must be in non-descending order.
The solution set must not contain duplicate subsets.

> For example,
If S =[1,2,3], a solution is:

> [
  [3],
  [1],
  [2],
  [1,2,3],
  [1,3],
  [2,3],
  [1,2],
  []
]

题意：给定一个包含不同整数的集合S，求出所有可能的不同的子集。

## 题解
这道题用动态规划解，因为要求出所有的子集，用dfs递归求解。暴力解法就是枚举，枚举所有的可能的情况，从每个位置开始，分别往后0个，1个...直到最后一个字符的情况。当长度固定了，接下来从每个位置开始查找序列就是dfs的过程：
``` java
ArrayList<ArrayList<Integer>> res=new ArrayList<>();
    public ArrayList<ArrayList<Integer>> subsets(int[] S) {

        if (S.length==0)
            return res;

        Arrays.sort(S);//先排个序
        ArrayList<Integer> cur=new ArrayList<>();
        for (int j=0;j<=S.length;j++)//循环子串的长度，从0到S.lengh
            trace(S,j,0,cur);
        return res;
    }

    /**
     *
     * @param S
     * @param k 子串的长度
     * @param start 数组S的下标开始位置
     * @param cur
     */
    private void trace(int[] S,int k,int start,List<Integer> cur){

        if (k<0)
            return;
        else if (k==0)//添加空
            res.add(new ArrayList<>(cur));
        else {
            for (int i=start;i<S.length ;i++){
                cur.add(S[i]);
                trace(S,k-1,i+1,cur);//继续添加子串的下一个数
                cur.remove(cur.size()-1);//回溯
            }
        }
    }
```
这道题的输入S中的数都是不同的，如果有相同的话，那么就会出现很多重复，解决办法是在一次dfs里边设置一个集合保存所有已经考虑过的数字，结果使用set保存。