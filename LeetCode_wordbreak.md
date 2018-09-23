[TOC]
## 题目
* 题目：word-break-ii
* 题目类型：动态规划

> Given a string s and a dictionary of words dict, add spaces in s to construct a sentence where each word is a valid dictionary word.
Return all such possible sentences.
For example, given
s ="catsanddog",
dict =["cat", "cats", "and", "sand", "dog"].
A solution is["cats and dog", "cat sand dog"].

题意：给定一个字符串s，一个单词字典dict，对s进行分割，使得分割后的每个单词都在字典中存在。

## 题解
这道题是动归题，又要输出中间的情况，有递归(dfs)和动归的解法。
#### 递归
下面的代码是从前往后dfs，**结果超时**：
``` java
	public ArrayList<String> res=new ArrayList<String>();//全局存放结果
    public  ArrayList<String> wordBreak(String s, Set<String> dict) {
        int dict_maxlen=-1;
        for (String d:dict)//找到最大的长度，主要用于搜索剪枝
            if (d.length()>dict_maxlen)
                dict_maxlen=d.length();
        dfs(s,0,"",dict,dict_maxlen);
        return res;
    }
    /**
    * index为当前开始搜索位置，cur_string为当前的合格的字符串
    **/
    public void dfs(String s,int index,String cur_string,Set<String> dict,int dict_maxlen){

        if (index==s.length()){
            if (cur_string.length()>0){
                res.add(cur_string.trim());
                return;
            }
        }
        //从index位置开始往后遍历，当连续的字符子串是字典中的单词的时候，那么从子串尾后一位继续递归
        for (int i=index;i<s.length()&&(i-index)<dict_maxlen;i++){
            if (dict.contains(s.substring(index,i+1)))
                dfs(s,i+1,cur_string+" "+s.substring(index,i+1),dict,dict_maxlen);
        }
    }
```
然后尝试改成了从后往前的dfs解法，**通过**：
``` java
	public ArrayList<String> res=new ArrayList<String>();
    public void dfs(String s,int index,String cur_string,Set<String> dict,int dict_maxlen){
        if(index==0){
            if(cur_string.length()>0)
                res.add(cur_string.substring(0,cur_string.length()-1));
        }
        for(int i=index;i>=0&&index-i<=dict_maxlen;i--){
            if(dict.contains(s.substring(i,index))){
                dfs(s,i,s.substring(i,index)+" "+cur_string,dict,dict_maxlen);
            }
        }
    }
    public  ArrayList<String> wordBreak(String s, Set<String> dict) {
        int dict_maxlen=-1;
        for (String d:dict)
            if (d.length()>dict_maxlen)
                dict_maxlen=d.length();
        dfs(s,s.length(),"",dict,dict_maxlen);
        return res;
    }
```
从前往后的超时，但是从后往前的dfs却可以ac，只能说**从前往后的时候回溯次数比较多，终止递归的位置太靠后**，所以从后往前的搜索可以提前终止，算法快很多。网上百度了下，说是有这么个样例：
s="aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaab"
dict=["a","aa","aaa","aaaa","aaaaa","aaaaaa","aaaaaaa","aaaaaaaa","aaaaaaaaa","aaaaaaaaaa","aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaab"]
从前往后回溯的时候，只有搜索到最后的b的时候才会回溯，那么往前回溯后根据for循环需要再增加一位字符子串的长度，直到回溯到当前位置右边的子串为aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaab的时候才会生成一种结果。**总的来说，从右往左比从左往右减少的是对aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaab长度的a的字符串的递归回溯。**
所以，不管上边的从左往右还是从右往左的这种递归dfs的思路很依赖测试数据，不可取，虽然侥幸过了，所以需要换一种思路，下面是[leetcode的高票dfs思路](https://leetcode.com/problems/word-break-ii/discuss/44167/my-concise-java-solution-based-on-memorized-dfs)：
``` java
	public List<String> dfs(String s, Set<String> dict, HashMap<String,ArrayList<String>> map){
        if (map.containsKey(s))//去重，防止前面已经出现过的序列再次出现
            return map.get(s);

        ArrayList<String> res=new ArrayList<>();

        if (s.length()==0){//判断终止条件，s为空了以后说明当次查询终止
            res.add("");
            return res;
        }
        for (String word:dict){//遍历字典的每个单词
            if (s.startsWith(word)){//如果字符串s以某个单词开头，那么继续递归子串
                List<String> sublist=dfs(s.substring(word.length()),dict,map);
                for (String sub:sublist)
                    res.add(word+(sub.isEmpty()?"":" ")+sub);
            }
        }
        map.put(s,res);//设置当前剩余子串的合格序列
        return res;
    }

    public ArrayList<String> wordBreak(String s, Set<String> dict) {
        return (ArrayList<String>) dfs(s,dict,new HashMap<>());
    }
```
leetcode的解法思路是，使用一个HashMap来保存子字符串和合格的单词划分组合，key为字符串，值为单词划分组合，最终返回key为s的列表。dfs遍历的时候遍历的是单词字典，若剩余字符串以某个单词开头，那么递归计算子字符串的组合列表，将当前的单词加到前面形成一种解法。
**注意：牛客网上测试样例输出顺序的原因，这个解法没通过，不打紧！**

#### 动态规划+dfs
这题其实动态规划不如dfs递归，写起来也不似标准的动态规划的两层for，网上有一种动态规划+dfs的组合解法：
``` java
	ArrayList<String> res=new ArrayList<>();//保存最终结果
    ArrayList<String> mid=new ArrayList<>();//保存中间遍历结果
    public ArrayList<String> wordBreak(String s, Set<String> dict) {

        if (s.length()<=0)
            return res;

        boolean[][] dp=new  boolean[s.length()][s.length()];//一维表示从位置i开始，二维表示单词的长度-1

//        计算从某个位置开始往后的一个子串是否能组成一个单词
        for (int i=0;i<s.length();i++){
            for (int j=i;j<s.length();j++){
                if (dict.contains(s.substring(i,j+1)))
                    dp[i][j-i]=true;
            }
        }

        find(s,s.length()-1,dp);
        Collections.reverse(res);//因为本题顺序原因，倒序
        return res;
    }

    /**
     *
     * @param s 字符串
     * @param len 字符串最后一个字符的位置
     */
    private void find(String s,int len,boolean[][] dp){
        if (len>=0){
            for (int i=0;i<=len;i++){//起始位置
                if (dp[i][len-i]){//如果从当前位置i到len的子串是个单词，因为是从后往前判断，这里的len-i表示单词长度-1
                    mid.add(s.substring(i,len+1));//添加从i到len的单词
                    find(s,i-1,dp);
                    mid.remove(mid.size()-1);
                }
            }
            return;
        }else {
            StringBuilder str=new StringBuilder();
            for (int i=mid.size()-1;i>=0;i--){//因为是从后往前加的，所以这里要从后边开始遍历
                str.append(mid.get(i));
                if (i>0)
                    str.append(" ");
            }
            res.add(str.toString());
        }
    }
```
这种解法用到dp数组，主要作用是提前记录单词的起始位置和长度，起到一点剪枝作用，最终还是要dfs，但是起始dfs的时候还是要挨个位置遍历，所以起始剪枝效果很小。这种方法看看就好。
