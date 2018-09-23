[TOC]
## 什么是ThreadLocal？
首先，ThreadLocal不是专门用来解决线程共享问题的，它的主要作用是为线程提供保持对象的方法，避免参数传递和提供一种方便的对象访问的方式。
同时，ThreadLocal也为解决多线程并发问题提供的一种新思路。ThreadLocal是线程的局部变量，线程私有。在多线程访问一个共享变量的时候，很容器引起线程安全问题，这时候往往是采用同步加锁的方式防止变量的访问出现问题，比如使用synchronized加锁。使用加锁的话，同一时间只能有一条线程对共享变量进行访问，当线程的操作时间不一定时，会严重影响性能。然而，有另一种方法可以控制对共享变量的访问，那就是为每一个线程提供一个共享变量的副本，这是一种以空间换性能的方式。这样，每个线程都有自己的本地变量，线程之间对变量的访问不会有影响。

## ThreadLocal应用场景
ThreadLocal常用在解决数据库链接，Session管理，客户端连接管理等场景。
比如，在多个客户端访问服务器的时候，服务器为客户端开多条线程，这个时候，对于不同的用户，线程会保存一份本地变量信息，像用户自身的信息等，这样在不同的用户访问一些页面的时候，会显示自己的信息。
再比如，在Spring AOP里边，现有的类的方法的参数是不可变的了，那么可以通过在本地线程中存储附加的参数信息，然后在调用方法的时候使用。

## ThreadLocal实现原理
首先来看一下Thread线程类：
``` java
public class Thread implements Runnable {
	ThreadLocal.ThreadLocalMap threadLocals = null;
}
```
在Thread类里边定义了threadLocals变量，这个变量是ThreadLocalMap类型的，而ThreadLocalMap是ThreadLocal的一个静态内部类。

#### ThreadLocalMap的基本结构
接下来观察下ThreadLocal.ThreadLocalMap静态内部类：
``` java
	static class ThreadLocalMap {
    	//静态内部类，是WeakReference的子类，是对ThreadLocal的引用，也就是说Entry的键是ThreadLocal
    	static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        private static final int INITIAL_CAPACITY = 16;//默认数组大小16
        private Entry[] table;
        private int size = 0;
        private int threshold;
        //ThreadLocalMap采用是懒加载的，初始只创建一个实体
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);//threshold设为len*2/3，也就是容量的三分之二
        }
    }
```
上述是ThreadLocalMap静态内部类的实体结构和基本参数变量，ThreadLocalMap内部维持一个Entry数组，Entry是WeakReference的子类，WeakReference在每次垃圾回收的时候都会被回收。Entry数组的初始容量为16，阈值threshold大小为初始容量的2/3，。ThreadLocalMap采用懒加载，初始对于整个Entry数组只初始化一个Entry实体。

#### ThreadLocalMap的getEntry()方法
``` java
		private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)//直接在数组中找到
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
   //若没有直接在数组中找到要找的实体
   		private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;
            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)//如果key是空，可能被回收
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
        //从当前位置往后直到下一个null之前，丢弃已经废弃的实体，将老的实体重新放到新的位置
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // 如果新的位置已经有实体，从该实体往后找到第一个null的位置放置
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```
上述代码讲述了getEntry()方法的整个流程，这个方法首先会从数组中查找是否有要找的实体，若要找的位置本身实体就不存在，返回null。如果是key不存在，说明可能被垃圾回收了，此时遍历从当前位置到下一个null之前的实体，废弃key已经为null的实体，将旧的实体重新放到新的位置。

#### ThreadLocalMap的set()方法
``` java
	private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            //i的位置不为null，需要遍历数组
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {//已存在，更新值就好
                    e.value = value;
                    return;
                }

                if (k == null) {//如果key为空
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            //i处位置为null，那么直接创建新的实体插入
            tab[i] = new Entry(key, value);
            int sz = ++size;
            //如果数组当前实体数量超过了threshold，那么需要扩容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
    private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // 从staleSlot位置往前找到第一个已经被垃圾回收的旧的Entry
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            // 从staleSlot往后找，直到第一个Entry为null的位置之前
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                // 当找到key相同的实体后，更新实体的值，然后将这个实体和staleSlot位置的实体交换，使位置正确
                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    //清理废弃的位置
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // 如果当前位置已经被回收，那么设定为待废弃的位置
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // 如果都没有找到key相同的位置，那么直接在当前位置创建新的实体
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
        //清除被gc的实体
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }
```
上述代码的流程如下，首先会用哈希获得要插入的位置，从当前位置开始遍历，如果当前位置不是null，那么判断实体的key是否是null。若key等于插入的key，那么直接更新值；如果key为null，那么进入replaceStaleEntry()函数。这个函数首先从当前位置往前到null之间找到第一个被垃圾回收的实体，然后再从当前位置往后遍历，同样，若找到key等于插入key的实体的话那么更新值，**注意，这时要把更新后的找到的这个实体和原先的实体交换一下，然后回收原先的实体**;如果没有找到key相同的实体，那么直接在staleSlot位置插入新的实体，**注意，同样会回收废弃的实体，这里size不会变化**。
cleanSomeSlots()方法作用是清理在两个参数位置之间的所有的被gc的实体，但是并不是所有的都遍历，长度每次缩小一倍，这里还是会调用expungeStaleEntry()方法。
接下来说一下扩容方法resize()：
``` java
	private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }
    //正式扩容
    private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;//长度为原来的两倍
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {//丢弃废弃的实体
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }
```
扩容方法resize()将数组的大小扩充为原来的两倍，再扩充的过程中同时会辅助进行垃圾回收。

#### ThreadLocalMap的remove()方法
``` java
	private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```
remove()方法比较简单，也是遍历数组，若找到key相等的实体，会清除掉对应实体，然后还是会调用expungeStaleEntry()方法。

#### ThreadLocal的setInitialValue()方法
``` java
	private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    protected T initialValue() {
        return null;
    }
```
方法比较简单，会从当前线程获取ThreadLocalMap，若没有map的话，那么会创建新的ThreadLocalMap，如果有的话，那么插入键-值，没有的话新建。initialValue()该方法可以在子类里覆盖。