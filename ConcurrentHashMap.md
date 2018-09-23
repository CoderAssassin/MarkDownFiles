# ConcurrentHashMap理解
[TOC]
这几天看了一些关于ConcurrentHashMap的文章，加深了理解，写篇简短的总结记录一下，免得下一次又得再找别人的文章。
ConcurrentHashMap在JDK 1.6、1.7和1.8有差别，尤其是JDK 1.8版本变化较大，JDK 1.6、1.7版本都是采用的分段锁(Segment (继承ReentrantLock))来实现对部分数据加锁，而在1.8中，进行了重新设计，加入了Node、TreeNode和TreeBin等数据结构。
## 预知识：HashMap
基本数据结构使用Entry，创建一个Entry数组，每个位置处是一个链表。
``` java
static class Entry<K,V> implements Map.Entry<K,V> {
        final K key;
        V value;
        Entry<K,V> next;
        int hash;
        }
        ```
当出现哈希碰撞的时候，采用链地址法解决冲突。
1.7版本使用的是链表，1.8版本使用的是红黑树，当节点个数大于8个的时候，转换为红黑树。
> 允许一个key为null，value为null的Entry，放到位置0处。

## Jdk 1.6、1.7的设计
### 总体设计
![concurrentHashMap1](https://github.com/CoderAssassin/markdownImg/blob/master/concurrentHashmap.png?raw=true "concurrentHashMap1.6/1.7")
采用分段锁的设计，同一个分段内的数据存在竞争，不同分段内的数据不存在竞争，并没有对整个Map数组进行加锁。ConcurrentHashMap存储有多个分段锁，每个分段锁内部有一个数组，数组的每个元素是HashEntry，从数组的元素的next指针找下去形成一条链表。
``` java
static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
        }
        ```
> key和value不能为null，若为null说明当前线程没有处理完而被其他线程看到。

### 重要参数
* ConcurrentLevel(并发度)
并发度就是锁的个数，一经指定不可改变。ConcurrentHashMap的默认并发度为16，如果创建的时候用户自定义并发度，那么创建后的并发度为大于等于用户并发度的最小2次幂值(比如自定义17最后是32)。之所以取2的幂值是因为可以直接进行位运算来扩容。
并发度太小，竞争大；并发度太大，CPU cache命中率下降。
* initialCapacity
决定整个ConcurrentHashMap的初始容量，默认为16，会均分到每个Segment的数组。
* loadfactor
负载因子，默认为0.75，是给每个Segment内部数组用的，现在的HashEntry数组的长度*loadFactor为threshold，扩容点。
* segmentShift、segmentMask
segmentShift表示并发度从高位到最高位1(包含)的位数，segmentMask表示剩余的位全为1的数，这样只要和segmentMask做与运算就可以定位到分段锁数组的下标位置。因为分段锁数组大小最大16位(65535)，所以segmentShift最大也是65535。
* segmentMask
哈希掩码，为ssize-1，通过(hash>>>segmentShift)&segmentMask来定位在某个分段锁自身的数组里的下标。

### 初始化
``` java
	public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
        
        int sshift = 0;//最小二次幂的最左边的1右边的位数
        int ssize = 1;//最小二次幂的值
        // 计算大于等于初始化并发度的最小二次幂值
        while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
        }
        this.segmentShift = 32 - sshift;//从最高位1开始到最左边的位数
        this.segmentMask = ssize - 1;//分段掩码

        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;

        // initialCapacity 是设置整个 map 初始的大小，
        // 这里根据 initialCapacity 计算 Segment 数组中每个位置可以分到的大小
        // 如 initialCapacity 为 64，那么每个 Segment 或称之为"槽"可以分到 4 个
        int c = initialCapacity / ssize;
        if (c * ssize < initialCapacity)//因为除的时候是取下整，所以这里需要调整一下
            ++c;

		//容量从2开始，这里是MIN_SEGMENT_TABLE_CAPACITY=2是为了避免延迟创建segement后使用的时候立马就要resize()
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)//计算大于等于每个段容量c的最小二次幂
            cap <<= 1;

        // 创建 Segment 数组，
        // 并创建数组的第一个元素 segment[0]
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        // 往数组写入 segment[0]
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
	}
```
上述代码是ConcurrentHashMap的构造函数。这里进行初始化。这里有几个参数，**segmentShift**表示32位的并行度concurrencyLevel从最高位1开始到最高位的位数；**segmentMask**表示concurrencyLevel的掩码，也就是剩余的位每一位都是1；这两个参数主要是用来定位Segment数组下标的。最后是创建Segment数组，创建第一个Segment。**注意：对于Segment的创建采用的是延迟加载的方式，初始只创建第一个Segment**。

### put方法
``` java
	public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        // 1. 计算 key 的 hash 值
        int hash = hash(key);
        // 2. 根据 hash 值找到 Segment 数组中的位置 j
        //    hash 是 32 位，无符号右移 segmentShift(28) 位，剩下高 4 位，
        //    然后和 segmentMask(15) 做一次与操作，也就是说 j 是 hash 值的最后 4 位，也就是槽的数组下标
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject
             (segments, (j << SSHIFT) + SBASE)) == null)
            s = ensureSegment(j);//延迟初始化新的Segment
        // 3. 插入新值到 槽 s 中
        return s.put(key, hash, value, false);
    }
```
上述代码主要流程如下：
* 根据key计算hash值，类似HashMap，将高16位与低16位做异或运算
* 根据hash值右移segmentShift位后与segmentMask做与运算得到Segment数组的索引下标。**HashMap使用的是低位定位数组索引下表，而ConcurrentHashMap使用的是高位定位数组索引下标。**
* 判断Segment是否初始化，没有的话调用**ensureSegment()**方法初始化Segment数组

接下来看一下ensureSegment()方法：
``` java
	private Segment<K,V> ensureSegment(int k) {
        final Segment<K,V>[] ss = this.segments;
        long u = (k << SSHIFT) + SBASE; // raw offset
        Segment<K,V> seg;
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
            // 使用到第一个Segment设置的参数来初始化接下来的Segment
            Segment<K,V> proto = ss[0];
            int cap = proto.table.length;
            float lf = proto.loadFactor;
            int threshold = (int)(cap * lf);

            HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
            if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                == null) { // 再次检查一遍该槽是否被其他线程初始化了。

                Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
                // 使用 while 循环，内部用 CAS，当前线程成功设值或其他线程成功设值后，退出
                while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                       == null) {
                    if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                        break;
                }
            }
        }
        return seg;
    }
```
上述代码是创建新的Segment的过程，**这里可能有多条线程同时进行初始化，但是最终只有一条线程能够初始化成功**，可以看到代码中有多处判断是否已经初始化完毕。上述代码中用到了第一个Segment的各个参数来初始化一个新的Segment。
初始化完毕后，那么就是Segment内部的HashEntry数组的put插入了，代码如下：
``` java
	final V put(K key, int hash, V value, boolean onlyIfAbsent) {
        // 尝试获取锁，tryLock()失败会继续调用scanAndLockForPut()，这个方法里是先遍历链表看有没有节点，没有的话再创建一个新的节点，然后再尝试获取锁
        HashEntry<K,V> node = tryLock() ? null :
            scanAndLockForPut(key, hash, value);
        V oldValue;
        try {
            // 这个是 segment 内部的数组
            HashEntry<K,V>[] tab = table;
            // 计算数组索引下标
            int index = (tab.length - 1) & hash;
            // 下标处的HashEntry实体
            HashEntry<K,V> first = entryAt(tab, index);

            // 遍历实体链表
            for (HashEntry<K,V> e = first;;) {
                if (e != null) {//若已经有实体
                    K k;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {//若对象已存在，找到并更新已有对象的值
                        oldValue = e.value;
                        if (!onlyIfAbsent) {
                            // 覆盖旧值
                            e.value = value;
                            ++modCount;
                        }
                        break;
                    }
                    // 继续顺着链表走
                    e = e.next;
                }
                else {//如果没有实体
                    if (node != null)//node指向第一个节点
                        node.setNext(first);
                    else//重新创建个实体
                        node = new HashEntry<K,V>(hash, key, value, first);
<
                    int c = count + 1;
                    // 判断是否需要扩容
                    if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                        rehash(node);
                    else
                        setEntryAt(tab, index, node);//将node插入链表头
                    ++modCount;
                    count = c;
                    oldValue = null;
                    break;
                }
            }
        } finally {
            unlock();
        }
        return oldValue;
    }
```
上述代码是Segment内部HashEntry数组的put操作，因为是并发，所以这里要加锁，只允许一个线程执行put操作，先根据hash定位到数组的下标，然后取得当前下标处的HashEntry实体，如果为空，那么插入新的node，如果非空，那么插入链表头，**在插入之前，需要判断是否需要扩容**。
上述代码在tryLock()获取锁失败会调用scanAndLockForPut()进行加锁，具体如下：
``` java
	private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
        HashEntry<K,V> first = entryForHash(this, hash);
        HashEntry<K,V> e = first;
        HashEntry<K,V> node = null;
        int retries = -1;

        // 循环获取锁
        while (!tryLock()) {
            HashEntry<K,V> f;
            if (retries < 0) {
                if (e == null) {
                    if (node == null) // speculatively create node
                        // 进到这里说明数组该位置的链表是空的，没有任何元素
                        // 当然，进到这里的另一个原因是 tryLock() 失败，所以该槽存在并发，不一定是该位置
                        node = new HashEntry<K,V>(hash, key, value, null);
                    retries = 0;
                }
                else if (key.equals(e.key))
                    retries = 0;
                else
                    // 顺着链表往下走
                    e = e.next;
            }
            // 重试次数如果超过 MAX_SCAN_RETRIES（单核1多核64），那么不抢了，进入到阻塞队列等待锁
            //    lock() 是阻塞方法，直到获取锁后返回
            else if (++retries > MAX_SCAN_RETRIES) {
                lock();
                break;
            }
            else if ((retries & 1) == 0 &&
                     // 这个时候是有大问题了，那就是有新的元素进到了链表，成为了新的表头
                     //     所以这边的策略是，相当于重新走一遍这个 scanAndLockForPut 方法
                     (f = entryForHash(this, hash)) != first) {
                e = first = f; // re-traverse if entry changed
                retries = -1;
            }
        }
        return node;
    }
```
上述代码是这样的，申请锁时，先tryLock()，在该过程中对其hashCode对应的HashEntry的链表遍历，如果没找到key相同实体那么重新创建一个，当tryLock()一定次数后用lock()申请锁，这样做目的是使链表被CPU cache缓存。并发时put(),rehash(),remove()等导致链表头结点变化，那么需要重新扫描链表。
接下来看一下扩容方法rehash()：
``` java
private void rehash(HashEntry<K,V> node) {
           HashEntry<K,V>[] oldTable = table;
           int oldCapacity = oldTable.length;
           int newCapacity = oldCapacity << 1;
           threshold = (int)(newCapacity * loadFactor);
           HashEntry<K,V>[] newTable =
               (HashEntry<K,V>[]) new HashEntry[newCapacity];
           int sizeMask = newCapacity - 1;
           for (int i = 0; i < oldCapacity ; i++) {
               HashEntry<K,V> e = oldTable[i];
               if (e != null) {
                   HashEntry<K,V> next = e.next;
                   int idx = e.hash & sizeMask;
                   //数组下标处只有一个元素
                   if (next == null)
                       newTable[idx] = e;
                   else { //实体下标处是个链表
                       HashEntry<K,V> lastRun = e;
                       int lastIdx = idx;
                       //从头到尾遍历，找到hash值不同的最后的位置，反过来说，从lastRun处的元素开始往后所有的元素的hash都是相等的，因此接下来的for循环是处理lastRun前面的元素，将它们的位置重新安放
                       for (HashEntry<K,V> last = next;
                            last != null;
                            last = last.next) {
                           int k = last.hash & sizeMask;
                           if (k != lastIdx) {
                               lastIdx = k;
                               lastRun = last;
                           }
                       }
                       newTable[lastIdx] = lastRun;//现将不变的链表部分存入新的数组
                       //将lastRun往前的链表里的实体节点重新hash到lastIndex所在的链表里，然后在头部插入新的元素
                       for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                           V v = p.value;
                           int h = p.hash;
                           int k = h & sizeMask;
                           HashEntry<K,V> n = newTable[k];
                           newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                       }
                   }
               }
           }
           //扩容之后，将新的节点插入
           int nodeIndex = node.hash & sizeMask;
           node.setNext(newTable[nodeIndex]);
           newTable[nodeIndex] = node;
           table = newTable;
       }
```
扩容是针对HashEntry数组的，将数组长度扩大为原来的两倍。扩容的时候，首先是遍历旧的HashEntry数组，在处理具体某个索引下标处的元素e的时候做如下两件事：

* 根据e的hash&sizeMask计算下标索引，若e并不是链表，那么直接在新数组插入元素e；若e是链表，做如下两件事：
	* 第一次循环遍历链表，找到第一个hash值不再变化的元素下标lastIdx
	* 第二次循环遍历链表，从元素e遍历到lastIdx之前，在新数组的对应位置的头部插入新的元素
* 插入新的元素

### get方法
``` java
	public V get(Object key) {
        Segment<K,V> s;
        HashEntry<K,V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
}
```
get()方法相比于put方法简单很多，主要分为如下几步：
* 根据hash值定位到具体的Segment
* 遍历Segment的HashEntry链表，找到key相等或者hash值和key值一样的实体，找不到的话返回null。

### get/containKey
通过getObjectVolatile()原子读获得对应链表进行遍历查找。因为是并发的，所以可能得到的结果是过时的数据，所以ConcurrentHashMap具有弱一致性。

### size/containsValue
原理：不加锁循环，遍历所有Segment，获得modcount和，若两次计算得到的一样大小，说明没有其他线程更改ConcurrentHashMap，返回值；否则对所有Segment加锁计算。

## JDK 1.8的设计
![concurrentHashMap1.8](https://github.com/CoderAssassin/markdownImg/blob/master/concurrentHashMap1.png?raw=true "concurrentHashMap1.8")

去掉Segment，采用Nodpe，TreeNode，TreeBin等数据结构，底层采用数组/链表/红黑树的设计。

#### 重要参数
* sizeCtl(控制标识符)
	* -1
正在初始化。
	* -N
有N-1个线程在扩容。
	* 正数或0
还没有初始化，值表示下一次扩容大小。

#### 新的数据结构
* Node
类似于HashEntry，位于第一层次。增加了find()辅助get()。
* TreeNode
当需要将链表转换成红黑树的时候，先将Node包装成TreeNode，带有next指针。
* TreeBin
转化成红黑树时，对多个TreeNode进行封装，所以ConcurrentHashMap数组中存放的也就是TreeBin了，也是对应的红黑树的根。带有读写锁。
* ForwardingNode
连接两个数组节点，节点或者是Node或者是TreeBin，它的属性都为null，唯独hash为-1。

#### 初始化initTable
``` java
    public ConcurrentHashMap(int initialCapacity) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException();
        int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                   MAXIMUM_CAPACITY :
                   tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
        this.sizeCtl = cap;
    }
```
上面是构造函数，**初始化时，sizeCtl设置为1.5*initialCapacity+1**，然后对计算的结果取大于等于该值的最小二次幂值。

#### 扩容transfer
##### 第一步，构建nextTable
**单线程**，容量扩为原来的两倍。总体思想：遍历-复制。
* 若数组的当前位置为空，放一个forwardNode节点
* 若当前位置是Node节点，遍历，构造一个反序序列，尾节点放在新数组的i+n(n为原数组大小)位置，原来Node节点放到i位置。
* 若当前位置是TreeBin节点，也反序，然后放到i+n，原来放到i位置。
* 遍历后，设置sizeCtl为0.75*新容量。

##### 第二步，复制
![concurrentHashMap_Transfer](https://github.com/CoderAssassin/markdownImg/blob/master/concurrentHashMap3.jpg?raw=true)

**多线程**。遍历原数组，若遍历到forwardNode节点，那么跳过继续往下遍历。否则遍历对应的Node链表或者树。每处理完一个节点设置对应Node或者TreeNode/TreeBin的值为forwardNode，另一个线程看到forwardNode直接跳过往后继续。

#### put
``` java
	public V put(K key, V value) {
    	return putVal(key, value, false);
	}
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {//遍历整个Node数组
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0)//数组为空那么初始化数组
                tab = initTable();

            // 数组下标位置为null，
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                // 如果数组该位置为空，
                //    用一次 CAS 操作将这个新值放入其中即可，这个 put 操作差不多就结束了，可以拉到最后面了
                //          如果 CAS 失败，那就是有并发操作，进到下一个循环就好了
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            // hash 居然可以等于 MOVED，这个需要到后面才能看明白，不过从名字上也能猜到，肯定是因为在扩容
            else if ((fh = f.hash) == MOVED)
                // 帮助数据迁移，这个等到看完数据迁移部分的介绍后，再理解这个就很简单了
                tab = helpTransfer(tab, f);

            else { // 到这里就是说，f 是该位置的头结点，而且不为空

                V oldVal = null;
                // 获取数组该位置的头结点的监视器锁
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) { // 头结点的 hash 值大于 0，说明是链表
                            // 用于累加，记录链表的长度
                            binCount = 1;
                            // 遍历链表
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                // 如果发现了"相等"的 key，判断是否要进行值覆盖，然后也就可以 break 了
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                // 到了链表的最末端，将这个新值放到链表的最后面
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) { // 红黑树
                            Node<K,V> p;
                            binCount = 2;
                            // 调用红黑树的插值方法插入新节点
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                // binCount != 0 说明上面在做链表操作
                if (binCount != 0) {
                    // 判断是否要将链表转换为红黑树，临界值和 HashMap 一样，也是 8
                    if (binCount >= TREEIFY_THRESHOLD)
                        // 这个方法和 HashMap 中稍微有一点点不同，那就是它不是一定会进行红黑树转换，
                        // 如果当前数组的长度小于 64，那么会选择进行数组扩容，而不是转换为红黑树
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
        addCount(1L, binCount);
        return null;
    }
```
上述是put()函数的主要过程，
* 首先，大循环遍历整个Node数组，如果Node数组是空的话，那么初始化一个新的Node数组；
* 如果Node数组不为空，并且hash找到的数组的下标处为null，那么直接插入新的节点，结束大循环；
* 如果下标处节点的hash等于MOVED，表示是个Forwarding节点，那么调用HelpTransfer()进行数据迁移；
* 如果上述几种情况都不满足，那么说明节点处是链表或者红黑树
	* 若头节点的hash值大于等于0，说明是链表，那么遍历链表寻找是否有已存在节点，是的话更新值，否则**插入尾部**。
	* 如果hash值小于0的话，说明是红黑树，按照红黑树的方式更新现有节点或者插入新的节点
* 最后，当大循环结束，判断是否需要转化成红黑树，大于等于8个可能要转换，调用的**treeifyBin()**方法，但是当数组长度小于64的时候会进行数组扩容，而不是转换成为红黑树。

接下来，先看一下初始化Node数组的方法initTable()：
``` java
	private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            // sizeCtl<0说明已经在初始化
            if ((sc = sizeCtl) < 0)
                Thread.yield();
            //设置sizeCtl值为-1表明正在初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        //当sizeCtl为正的时候表明接下来初始化后的数组大小，如果是0的话那么是设置成默认
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        // sc变为0.75*n
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
                break;
            }
        }
        return tab;
    }
```
上述的初始化Node数组方法，主要是围绕**sizeCtl**来进行，若sizeCtl小于0说明已经有线程在初始化，那么就暂停；否则的话，设置数组的大小为sizeCtl或者默认16，然后创建新数组，最后设置sizeCtl=0.75*(原sizeCtl)。
接下来看一下**treeifyBin()**方法，看一下如何转为红黑树：
``` java
	private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            //若数组长度小于64，那么将数组扩大为原来的两倍，充分分配元素的位置
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                tryPresize(n << 1);
            //否则转换为红黑树
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                // 加锁
                synchronized (b) {

                    if (tabAt(tab, index) == b) {
                        // 下面就是遍历链表，建立一颗红黑树
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        // 将红黑树设置到数组相应位置中
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
	}
```
上述代码是在put()方法最后对链表的转换，若数组的长度小于64的话是不会对当前数组下标处的链表转换的。
接下来我们看一下数组扩容方法**tryPresize()**：
``` java
	//size是翻倍后的数组大小
    private final void tryPresize(int size) {
        // 1.5*size+1，取大于等于该值的最小二次幂值
        int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
            tableSizeFor(size + (size >>> 1) + 1);
        int sc;
        //sizeCtl大于等于0表示未初始化或者扩容
        while ((sc = sizeCtl) >= 0) {
            Node<K,V>[] tab = table; int n;
            //情况一：现有数组为空，那么初始化
            if (tab == null || (n = tab.length) == 0) {
                n = (sc > c) ? sc : c;//新数组的大小在c和sc中取大者
                if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {//令SIZECTL=-1，表示正在初始化
                    try {
                        if (table == tab) {
                            @SuppressWarnings("unchecked")
                            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                            table = nt;
                            sc = n - (n >>> 2);
                        }
                    } finally {
                        sizeCtl = sc;//设置sizeCtl为0.75*n，作为下一次扩容的大小
                    }
                }
            }
            //情况二：数组没有达到扩容的标准(threshold)或者无法继续扩容，那么直接退出
            else if (c <= sc || n >= MAXIMUM_CAPACITY)
                break;
            //情况三：正常扩容动作，这个判断条件是确定有没有扩容完毕
            else if (tab == table) {
                int rs = resizeStamp(n);//根据n不同生成不同的生成戳，用来作为不同次扩容的标记
                if (sc < 0) {
                    Node<K,V>[] nt;
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);
                }
                //还未初始化
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
            }
        }
    }
```
上述代码都是对sizeCtl值的操作，若是负数的话会进行初始化，接下来会更新sc，然后调用数据迁移**transfer()**方法进行扩容，关键是这个方法：
``` java
	private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;

        // 设置单线程和多线程情况下的步长stride，表示每次从后往前处理多少个数组节点
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range

        // 初始化数组
        if (nextTab == null) {
            try {
                // 大小变为原来的两倍
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;
        }

        int nextn = nextTab.length;

        // ForwardingNode除hash=Move外都是空，用来标记节点已经被处理，其他线程看到这个节点会直接调到下一个节点来处理
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);

        boolean advance = true;
        boolean finishing = false;

        // i 是位置索引，bound 是边界，注意是从后往前
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            //每一趟的遍历，只处理stride个节点
            while (advance) {
                int nextIndex, nextBound;
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {
                    i = -1;
                    advance = false;
                }
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    // sizeCtl变为原来的0.75倍
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }

                // sizeCtl减1表示一个线程完成了扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            //在数组null位置放入ForwardingNode
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            // 该位置处是一个 ForwardingNode，跳过该位置
            else if ((fh = f.hash) == MOVED)
                advance = true;
            else {
                // 加锁开始迁移
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        // hash大于等于0是链表，链表迁移和1.7的差不多
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            // 其中的一个链表放在新数组的位置 i
                            setTabAt(nextTab, i, ln);
                            // 另一个链表放在新数组的位置 i+n
                            setTabAt(nextTab, i + n, hn);
                            // 设置原数组该位置为ForwardingNode
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                        else if (f instanceof TreeBin) {//如果是红黑树，按红黑树的方式迁移
                            // 红黑树的迁移
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            // 如果一分为二后，节点数少于 8，那么将红黑树转换回链表
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;

                            // 将 ln 放置在新数组的位置 i
                            setTabAt(nextTab, i, ln);
                            // 将 hn 放置在新数组的位置 i+n
                            setTabAt(nextTab, i + n, hn);
                            // 设置原数组该位置为ForwardingNode
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }
```
上述代码关键是stride，这个变量的意义是每个线程对数组扩容的长度，是从数组后边往前看的。如果数组为空先初始化，若下标当前位置是null，那么放入ForwardingNode节点；若当前位置是链表，那么遍历链表；如果当前位置是红黑树，那么遍历红黑树。切分后一部分放在原来位置，还有一部分放在原来位置加上旧数组长度的位置处。这里的关键是ForwardingNode，这个节点表示当前位置已经处理，那么接下来的线程看到这个值后直接跳过去处理下一个。

#### get
``` java
	public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {//头节点就是查找的节点
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            else if (eh < 0)//在扩容或者当前位置是红黑树
                return (p = e.find(h, key)) != null ? p.val : null;

            // 遍历链表查找
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }
```
get()方法比较简单，首先根据hash定位到数组的位置，如果该位置为null，那么直接返回null；如果正好是要找的节点，那么返回值；如果hash<0说明在扩容或者是红黑树；如果都不满足说明是链表，进行遍历。

#### size
类似1.6/1.7，也是弱一致性，但是使用了一些变量和方法来提高一致性。

## Reference
1. [http://www.importnew.com/22007.html](http://www.importnew.com/22007.html)
2. [http://ifeve.com/concurrenthashmap/](http://ifeve.com/concurrenthashmap/)
3. [http://www.importnew.com/28263.html](http://www.importnew.com/28263.html)