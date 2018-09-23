[TOC]
## String的subString()方法
``` java
subString(int beginIndex,int endIndex);
subString(int beginIndex);
```
该方法是获取String字符串的从beginIndex(包含)到endIndex(不包含)之间的或者从beginIndex(包含)开始到结尾的子字符串。乍看之下，这个函数返回的是一个子字符串，不知道内部原理的情况下，我们可能会认为返回的是一个新的String，值为子字符串，然而真实情况却并不是这样的。
在subString内部的实现源码如下：
``` java
public String substring(int beginIndex, int endIndex) {
        if (beginIndex < 0) {
            throw new StringIndexOutOfBoundsException(beginIndex);
        }
        if (endIndex > count) {
            throw new StringIndexOutOfBoundsException(endIndex);
        }
        if (beginIndex > endIndex) {
            throw new StringIndexOutOfBoundsException(endIndex - beginIndex);
        }
        return ((beginIndex == 0) && (endIndex == count)) ? this :
            new String(offset + beginIndex, endIndex - beginIndex, value);//关键地方
}
```
上述代码中，最后创建的String，下面是它的构造函数：
``` java
String(int offset, int count, char value[]) {
        this.value = value;
        this.offset = offset;
        this.count = count;
}
```
offset是String字符串在底层数组的起始偏移地址，这里传入的是原来字符串的偏移地址+子字符串的开始位置=子字符串在底层数组的开始位置，传入的第二个参数表示子字符串的长度，**最后将原来数组value的引用直接作为subString函数的参数传入**，也就是说，**subString函数其实并没有创建新的字符串对象**，而是引用的原字符串对象的底层数组，通过起始偏移地址和长度来确定子字符串。
因此，在对长String对象调用subString的话，其实内存中还是存在长字符串的，并不会被垃圾回收。可以用`new String(str.subString(0,10));`来对子字符串创建堆中的新的String对象，gc会对原字符串进行回收。
> 参考博客：[你了解Java中String的substring函数吗？](https://www.cnblogs.com/tedzhao/archive/2012/07/31/Java_String_substring.html)

## String的split()方法
split()方法两种，分别是1个参数的和两个参数的，具体如下：
``` java
public String[] split(String regex) {
        return split(regex, 0);
    }
public String[] split(String regex, int limit);
```
从上面两个split的方法可以看出，一个参数的split方法最终调用的是带两个参数的split方法，传入的第二个参数是0。那么这第二个参数limit有什么用呢？
limit表示的是最后产生的结果数组的元素个数，主要分3种情况：

* limit为负数：那么最后产生的数组的长度无限制，也就是说，模式会匹配无数次。
* limit=0：最后产生的数组的长度也是没有限制的，但是，和负数情况不同的是，数组**会将尾部的空字符串抛弃掉**。
* limit为正数：那么最后产生的数组的长度就是limit，模式最多匹配limit-次；对于剩下的字符串统统塞到数组最后一个实体。

例如：
``` java
//代码1：
		String str="a,b,c,,,";
        String[] res=str.split(",");
        System.out.println(res.length);
        for (String r:res)
            System.out.println(r);
//输出
3
a
b
c
//代码2：
		String str="a,b,c,,,";
        String[] res=str.split(",",-1);
        System.out.println(res.length);
        for (String r:res)
            System.out.println(r);
//输出
6
a
b
c



//代码3：
		String str="a,b,c,,,";
        String[] res=str.split(",",2);
        System.out.println(res.length);
        for (String r:res)
            System.out.println(r);
//输出
2
a
b,c,,,
```
底层实现最终都是**返回一个新的String[]数组**。

## Spring的Join方法
Join方法的主要作用是在一个数组的连续的两个元素之间用分隔符分割，返回新的字符串。主要有两个**类方法**：
``` java
public static String join(CharSequence delimiter, CharSequence... elements);
public static String join(CharSequence delimiter,
            Iterable<? extends CharSequence> elements)；
```
方法实现一样，就是串的参数不一样，后者传入的是迭代器。以第一个为例，看下源码：
``` java
public static String join(CharSequence delimiter, CharSequence... elements) {
        Objects.requireNonNull(delimiter);
        Objects.requireNonNull(elements);
        // Number of elements not likely worth Arrays.stream overhead.
        StringJoiner joiner = new StringJoiner(delimiter);
        for (CharSequence cs: elements) {
            joiner.add(cs);
        }
        return joiner.toString();
    }
```
内部创建的StringJoiner对象，这里调用的构造函数只有分隔符，默认前缀和后缀都为空字符串""；重点来看一下add方法：
``` java
public StringJoiner add(CharSequence newElement) {
        prepareBuilder().append(newElement);
        return this;
    }
private StringBuilder prepareBuilder() {
        if (value != null) {
            value.append(delimiter);
        } else {
            value = new StringBuilder().append(prefix);
        }
        return value;
    }
```
在PrepareBuilder()里判断当前StringBuilder对象(也就是最终拼接的字符串)是否为空，为空的话加入前缀；否则append分隔符然后再append字符串；toString()里判断后缀是否为空，为空的话直接返回当前结果的新的String对象，否则，先append后缀，然后返回新的String对象。

