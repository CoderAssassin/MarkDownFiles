[TOC]
## 哈希表
首先说下HashTable，HashTable是一种以“键-值”映射的方式保存数据的数据结构，是一个散列表。底层采用**数组+链表**的方式来组织。
HashTable继承自Dictionary抽象类，并且实现了Map，Cloneable和Serialization三个接口。
HashTable是线程安全的，因为底层方法的实现很多加了Synchronized同步锁。

#### HashTable的数据结构
上边说了，HashTable采用的“数组+链表”的存储方式，数组和链表的每一个元素都是一个Entry，这个Entry是Map.Entry的一个实现，来看一下Entry的数据结构：
``` java
private static class Entry<K,V> implements Map.Entry<K,V> {
        final int hash;
        final K key;
        V value;
        Entry<K,V> next;
        }
```
hash表示当前节点的hash值，通过f(key)计算得到hash值，然后对数组长度取模得到；next指向链表的下一个Entry节点。

#### HashTable的重要参数
* initialCapacity：HashTable的**数组的初始容量，默认初始值为11**，在构建HashTable的时候可在构造函数中自定义，然后HashTable会根据传入的参数调节，具体的在接下来的源码阐述。
* loadFactor：HashTable的负载因子，**默认初始值为0.75**。loadFactor*initialCapacity计算得到的是thredhold，当当前HashTable的容量超过这个值的时候，那么会进行扩容。
* threshold：实体数量阈值，当当前哈希表的实体个数达到该值的时候，会进行扩容。
* modCount：哈希表的结构被修改的次数，主要是迭代时的fail-fast时起作用，判断是否再并发的时候有对哈希表结构的修改。
* count：HashTable的总的实体个数。

#### HashTable的构造函数
``` java
//默认的构造函数，初始容量11，负载因子0.75
public Hashtable()
//可自定义初始容量的构造函数
public Hashtable(int initialCapacity)
//自定义初始容量和负载因子
public Hashtable(int initialCapacity, float loadFactor);
//构造一个新的HashTable，存放Map的所有的实体
public Hashtable(Map<? extends K, ? extends V> t)
```
上面几个构造函数最后都是调用的public Hashtable(int initialCapacity, float loadFactor)这个构造函数，具体看下这个构造函数：
``` java
public Hashtable(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal Load: "+loadFactor);

        if (initialCapacity==0)//自定义0的话，那么只有初始容量设为1
            initialCapacity = 1;
        this.loadFactor = loadFactor;
        table = new Entry<?,?>[initialCapacity];//初始化数组大小为初始容量
        threshold = (int)Math.min(initialCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
    }
```
注意：HashTable不像ConcurrentHashMap那样会对自定义的初始化值进行处理，而是直接设定容量为自定义的初始值。

#### HashTable主要方法解析

* get方法

``` java
public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
```
首先，方法前面有**synchronized**同步锁标志，说明调用方法的时候需要加锁，这也是为什么HashTable是线程安全的。传入的参数是要获取的实体的key，先获得key对象的hashCode，再和最大整数做与运算，最后对数组的长度取模得到要查询的key在数组中的位置；接下来，遍历数组当前元素的链表判断是否有相等的实体，这里判断相等有两个条件，一是Entry的hash必须和key的hash相等，二是通过equals()方法判断是否key相等。不相等返回null。

* put()方法

``` java
public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> entry = (Entry<K,V>)tab[index];
        for(; entry != null ; entry = entry.next) {
            if ((entry.hash == hash) && entry.key.equals(key)) {
                V old = entry.value;
                entry.value = value;
                return old;
            }
        }

        addEntry(hash, key, value, index);
        return null;
    }
```

同样的，方法前加了synchronized表明调用put方法需要加锁。从代码看出，**HashTable的键和都值不能为null**。定位到数组中的元素位置和get()方法一样，put()方法设置实体值有两种情况：一是遍历链表发现当前链表中已经存在同样的key的实体，那么更新实体的值，返回旧实体值；二是不存在，那么添加新的实体，返回null。

看一下添加新实体的方法addEntry():
``` java
private void addEntry(int hash, K key, V value, int index) {
        modCount++;//结构修改一次，+1

        Entry<?,?> tab[] = table;
        if (count >= threshold) {//判断当前表的数组的元素数量是否大于threshold
            // Rehash the table if the threshold is exceeded
            rehash();//对表的实体进行rehash()

            tab = table;
            hash = key.hashCode();
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>) tab[index];
        tab[index] = new Entry<>(hash, key, value, e);//注意，在链表头插入新的节点
        count++;
    }
```
这里并不需要加锁，synchronized是个可重入锁，在调用put()方法的时候已经对调用put()方法的对象加锁，addEntry()也被加锁。
在这个方法里，先是modCound++表示数据结构被修改了一次。然后判断当前的HashTable中的表的实体数量是否超过了threshold，如果是，那么需要调用rehash()方法对整个表进行扩容。**注意，插入新的节点是在数组对应的位置的链表头插入新的元素**。

上述代码的关键的地方是rehash()函数，接下来看一下这个函数的具体实现：
``` java
protected void rehash() {
        int oldCapacity = table.length;
        Entry<?,?>[] oldMap = table;

        // overflow-conscious code
        int newCapacity = (oldCapacity << 1) + 1;//原来的容量的两倍+1
        if (newCapacity - MAX_ARRAY_SIZE > 0) {
            if (oldCapacity == MAX_ARRAY_SIZE)
                // Keep running with MAX_ARRAY_SIZE buckets
                return;
            newCapacity = MAX_ARRAY_SIZE;
        }
        Entry<?,?>[] newMap = new Entry<?,?>[newCapacity];

        modCount++;//同样修改了数据结构
        threshold = (int)Math.min(newCapacity * loadFactor, MAX_ARRAY_SIZE + 1);
        table = newMap;

//开始重新计算hash和实体的位置
        for (int i = oldCapacity ; i-- > 0 ;) {
            for (Entry<K,V> old = (Entry<K,V>)oldMap[i] ; old != null ; ) {
                Entry<K,V> e = old;
                old = old.next;

                int index = (e.hash & 0x7FFFFFFF) % newCapacity;
                e.next = (Entry<K,V>)newMap[index];
                newMap[index] = e;
            }
        }
    }
```
注意rehash的代码，**扩容是针对数组扩容的**。oldCapacity为旧数组容量，扩容后大小为**2*oldCapacity+1**，但是不能超过整数最大值-8，否则设为正数最大值-8。接着从后往前遍历数组的每个元素，然后从前往后遍历链表，重新计算hash值，**从链表的头部插入**。

* 几个contains方法
HashTable一共有3个contains方法，分别是contains()，containsKey()，containsValue()，基本用后边两个，containsValue()其实调用了contains()方法，只是为了区分key，所以重新定义了个函数。containsKey()方法和get()、put()方法里边一致。
``` java
//这个方法判断是否包含对应值的元素
public synchronized boolean contains(Object value) {
        if (value == null) {
            throw new NullPointerException();
        }

        Entry<?,?> tab[] = table;
        for (int i = tab.length ; i-- > 0 ;) {
            for (Entry<?,?> e = tab[i] ; e != null ; e = e.next) {
                if (e.value.equals(value)) {
                    return true;
                }
            }
        }
        return false;
    }
//其实就是调用的contains()方法
public boolean containsValue(Object value) {
        return contains(value);
    }
//判断是否包含key
public synchronized boolean containsKey(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return true;
            }
        }
        return false;
    }
```

* remove方法

``` java
public synchronized V remove(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        @SuppressWarnings("unchecked")
        Entry<K,V> e = (Entry<K,V>)tab[index];
        for(Entry<K,V> prev = null ; e != null ; prev = e, e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                modCount++;//结构也变化了
                if (prev != null) {
                    prev.next = e.next;
                } else {
                    tab[index] = e.next;
                }
                count--;//总的实体数减1
                V oldValue = e.value;
                e.value = null;
                return oldValue;
            }
        }
        return null;
    }
```
也是加同步锁，先是定位数组位置，然后遍历链表，找到key相等的，删除，这里因为同样是对结构修改，所以modCount+1.