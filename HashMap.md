[TOC]
## HashMap
HashMap和HashTable很像，也是采用“键-值”的方式存储数据，底层也是采用的数组+链表的数据结构。
HashMap集成自AbstractMap抽象类，AbstractMap是对Map接口的实现，里边定义了常用的contaninsKey()，containsValue()等方法；HashMap同样实现了Map<K,V>, Cloneable, Serializable接口。
**注意：下面的代码是在JDK 1.8下。**

#### HashMap和HashTable的区别
* 集成的抽象类不同，HashTable继承自Dictionary，HashMap继承自AbstractMap抽象类。
* HashMap是非线程安全的，HashTable是线程安全的，体现在HashTable的各个方法都有synchronized加锁。
* HashTable的键和值都不能为null，HashMap中可以有1个null键，若get()返回null，可能是没有该键，或者值为null，所以需要使用conta
## HashMap
HashMap和HashTable很像，也是采用“键-值”的方式存储数据，底层也是采用的数组+链表的数据结构。
HashMap集成自Abs
## HashMap
HashMap和HashTable很像，也是采用“键-值”的方式存储数据，底层也是采用的数组+链表的数据结构。
HashMap集成自AbsinsKey()来判断是否有某个键。
* 遍历方式不同，HashMap和HashTable都使用的是Iterator，但是因为历史原因，HashTable还使用了Enumeration方式
* JDK1.8之后HashTable才采用的fast-fail机制，HashMap一直有
* HashTable的默认初始容量为11，扩容方式为2*oldCapacity+1，若构造函数中设定初始值，那么数组长度就是该初始值；在HashMap中，默认初始容量为16，扩容方式为oldCapacity*2，如果构造函数中传入自定义的初始容量，那么会取大于等于该值的最小的2的幂值来作为初始容量。
* hash值计算方式不同，HashTable直接用对象的hashCode做与运算，再取模；HashMap用的是位运算，效率高。

#### HashMap的重要参数
* initialCapacity：HashMap初始容量，键-值对个数，**默认为16**。
* loadFactor：负载因子，**默认为0.75**。
* threshold：键-值对阈值，超过该值会进行扩容。
* size：已有键-值对的个数。

#### HashMap的基本的数据结构
``` java
static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Node<K,V> next;
        }
```
和HashTable不同的是名字，HashTable是Entry，这里是Node，其他都一样。

#### HashMap的构造方法
``` java
//默认构造方法，初始大小16，负载因子0.75
public HashMap();
//自定义初始容量的构造方法
public HashMap(int initialCapacity);
//自定义初始容量和负载因子的构造方法
public HashMap(int initialCapacity, float loadFactor);
//构造一个包含参数map对象的HashMap
public HashMap(Map<? extends K, ? extends V> m);
```
这里的构造方法和HashTable的构造方法基本一致，核心的还是public HashMap(int initialCapacity, float loadFactor)方法：
``` java
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```
和HashTable不同之处在于，若initialCapacity为0的话，HashTable会设定initialCapacity为1，但是这里没有限定；另外一个不同是，HashTable直接设定数组大小为initialCapacity，HashMap会调用tableSizeFor()将大小设定为大于等于输入的initialCapacity的最小2的幂值。
``` java
static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```
上述代码中间一系列的按位或的操作之后，最终是将cap的最左边的1往右的所有的0变成了1，这样的话再加1得到的就是大于等于cap的最小二次幂。比如，cap=1010(10)，那么减1就是1001(9)，或运算后就是1111(15)，最终加1为10000(16)。

**问题探讨**：上述代码或运算之前位移是1,2,4,8,16，移动的位数按两倍变化，这里什么玄机？其实这是一个很巧妙的算法来将右边的所有的位都置1，原理是这样的，假设有个二进制数10000001，共8位，那么右移1位后是01000000，或运算后是11000001，这样左边两位已经成1；接下来移动2位的意思是用左边的已经是1的两位将紧接着的后边的两位置1，变成了11110001；这时候，左边4位已经是1了，那么接下来用左边的4位将右边的紧接着的4位置1，变成了11111111；后边自然是使用这8位将后边的置1，但是后边没有数了，所以后边两步其实没啥用。

#### HashMap的主要方法
* get方法
``` java
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
```
这里只是个入口方法，主要实现还是再getNode()方法，在这之前，先要计算hash。如果最终返回null，那么直接返回null。看下hash()方法：
``` java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
这是个静态方法，如果key是null的话，那么直接返回0，说明如果有null键的话，null是插入到数组的下标为0的位置；否则的话，取key的hashCode和其高16位异或。
**问题探讨：这里采用的是对key的hashCode()返回结果的高16位和低16位做异或运算得到hash值，那么为什么要这么做呢？**
首先，hashCode()返回值是int型，范围是-2147483648到2147483648，大概40亿的映射空间，假设内存中有这么长的数组的话，数据的分布会是比较均匀的，但是实际情况是内存中不可能存的下这么大的数组。在HashMap中，实际的计算最终数组索引下标是通过**hash&(length-1)得到的，这里的length是数组的长度**，这样的话，只用到了低位来定位数组的索引下标，高位其实没有用到，这样的话冲突的概率很高。像我们初始化的时候数组的长度是16，只用到了hash的后4位，很容易冲突，因此就有了上边这段代码“扰动函数”。上述代码将高16位和低16位做异或运算，最后得到的低位随机性更高，因为掺杂了高位的部分特征，这样相当于将高位的信息也变相保存下来。(上边都是JDK 1.8的实现，但是1.7中是有进行4次扰动的，具体看1.7的代码，但是4次扰动后对减少碰撞的效果不大，考虑运行效率，所以1.8使用了一次)
接着具体来看一下getNode()方法：
``` java
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```
上述方法外层获得first引用指向要获取的key在数组中的位置，然后先判断first指向的对象的hash和key值是否和待获取的key值一样，是则直接返回first。如果不是的话，那么接下来就是遍历first的链表或者树，**因为JDK1.8之后，当链表节点个数大于等于8个的时候，会转为使用红黑树来存储节点，所以这里分为对链表的遍历和直接按树的查找方式查找**。

* put方法
``` java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```
这也只是个入口方法，重点看putVal()方法：
``` java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //需要建表
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //如果数组中要插入的位置为null，那么创建新的节点
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //要插入的位置就是数组对应位置的第一个节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //数组待插入位置是一颗红黑树
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            //数组待插入位置是链表
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                    //将新节点插入链表的尾部
                        p.next = newNode(hash, key, value, null);
                       //这里如果发现链表的节点数大于等于8，那么转为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //主要是设置新值
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }
```
上述putVal()方法的总体过程是这样的，首先判断表是否是空的，是的话要初始化。然后是两种情况，一是待插入的key在数组中的位置的元素为null，那么直接创建新的节点插入；二是不为null，此时分为3种情况，若数组对应元素的第一个节点就是key对应的节点，那么e设为该节点，若数组对应元素是棵树，那么按树的方式put，若是链表，那么遍历链表，找到是否有相同的节点，如果有那么e指向该节点，没有的话在尾部插入新的节点，然后e为null，对于e非null的情况(即原来就有对应的key的节点)，若onlyIfAbsent=false那么更新旧值。最终，判断是否需要扩容，是的话再进行扩容。
接下来，再看一下关键的初始化和扩容方法resize()：
``` java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
        //如果原来的容量超过了最大容量限制，那么不再扩容，返回旧的数组
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //如果新的容量是原来的两倍，那么阈值也直接变为原来的两倍就好
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        //如果原来的容量为<=0，但是阈值不是0，那么新的容量为原来的阈值
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
            //如果原来的容量和阈值都是0，那么说明是初始化过程，设为默认值
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //设定新的阈值
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //如果原来的表非空，那么遍历原来的表，将原来的节点重新分配到新的数组中
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {//数组对应位置有元素
                    oldTab[j] = null;
                    if (e.next == null)//遍历到尾部了，那么直接插入新的元素
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)//按红黑树的方式处理
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {//对链表进行遍历
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {//若e是老数组的节点
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {//若e的hash超过了老数组的界限
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {//放到老数组在新数组对应的位置
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {//放到新数组对应的位置
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
resize()方法主要是对数组进行初始化或者进行扩容，扩容后的数组大小为原来数组的大小的两倍。当原数组容量(oldCap)和阈值(threshold)都为0，说明要进行初始化；否则的话，进行扩容。扩容的大体过程是这样子的，对整个数组遍历，取出数组的元素，分为三种情况，一是当前位置只有一个元素，那么直接通过与新数组容量-1进行与运算，插入到新的数组中；二是当前节点为红黑树节点，那么按找树的方式处理；三是当前位置的元素是链表节点，那么开始遍历链表，这里将每个元素的hash和oldCap以及newCap与运算定位该元素在新的数组的位置，可能是在新数组对应原数组的数组位置，也可能是原数组位置加上oldCap。

* containsKey方法

``` java
public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }
```
这个方法很简单，和get()一样内部都调用了getNode()方法。

* containsValue方法

``` java
public boolean containsValue(Object value) {
        Node<K,V>[] tab; V v;
        if ((tab = table) != null && size > 0) {
            for (int i = 0; i < tab.length; ++i) {
                for (Node<K,V> e = tab[i]; e != null; e = e.next) {
                    if ((v = e.value) == value ||
                        (value != null && value.equals(v)))
                        return true;
                }
            }
        }
        return false;
    }
```
方法也很简单，遍历数组的每个元素，对于每个元素，遍历其每个元素，判断值是否相等即可。

* remove方法

``` java
public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
    }
//真正对节点的移除
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            //数组对应位置的第一个元素就是要删除的元素
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {
                if (p instanceof TreeNode)//说明数组对应位置是棵红黑树
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {//数组对应位置是个链表，那么接下来开始遍历链表
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)//按树的方式删除节点
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)//是第一个节点
                    tab[index] = node.next;
                else//按链表的方式删除
                    p.next = node.next;
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
    }
```
remove()方法的话，总的过程是这样的，首先，找到要删除的元素，也是分成三种方式来找，分别是数组对应位置，树节点以及链表中节点。然后也是按照三种方式删除节点。

#### Reference
* [https://blog.csdn.net/wangxing233/article/details/79452946](https://blog.csdn.net/wangxing233/article/details/79452946)