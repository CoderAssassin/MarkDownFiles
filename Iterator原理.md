[TOC]
## Iterator迭代器原理
按照官方文档的说法，Iterator是用来替换Enumeration的，和Enumeration相比起来有如下两点不同：
> 1.Iterator允许调用remove()方法从当前元素集合中删除元素
> 2.变动了一些方法名

想要使用Iterator需要实现这个接口，我们以ArrayList为例，平时使用的时候我们都是通过ArrayList对象的iterator()方法获得当前列表的Iterator对象，在ArrayList建立了**Itr**内部类来实现Iterator接口：
``` java
	public Iterator<E> iterator() {
        return new Itr();
    }
	private class Itr implements Iterator<E> {
        int cursor;       // 下一个将返回的元素下标
        int lastRet = -1; // 上一个返回的元素下标
        int expectedModCount = modCount;//fast-fail快速失败机制重要参数

        Itr() {}
   }
```
看一下上述代码，iterator()方法内创建Itr对象，而Itr是ArrayList内部的一个类，实现了Iterator接口，所以我们重点关注Itr内部类里边的实现，看一下常用的几个方法hasNext()、next()和remove()的实现：

#### hasNext()方法
``` java
	public boolean hasNext() {
            return cursor != size;
        }
```
hasNext()方法的实现很简单，这里的size是ArrayList对象的当前列表长度，也就是说，因为cursor下标是从0开始的，列表长度为size，所以cursor范围是[0,size-1]，上面这句代码的意思就是cursor没有超出列表长度就返回true，表示还有元素。

#### next()方法
``` java
	public E next() {
            checkForComodification();//检查是否发生并发修改异常
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;//浅拷贝了一份原数组的所有元素
            //如果要找的元素的下标超过原数组长度的界，说明原数组发生并发修改异常
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }
   //检查是否原列表发生了修改，是的话丢出并发修改异常
   final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
```
上述代码就是常用的next()方法的实现，在上边代码中，重点关注的是并发修改异常ConcurrentModificationException的抛出。当多个线程对容器进行操作的时候，就会发生并发修改异常。比如线程A对一个列表对象通过iterator来遍历集合，这个时候线程B对原列表进行**结构上的修改**，那么迭代就会出错。这个错误在平时使用for对列表进行循环遍历的时候很容易出现，这个时候若在for循环内部调用add()或者remove()方法，并且for循环判断条件是小于等于列表长度，那么就会发生循环混乱，甚至有可能发生无限循环(每次都add一个新的元素)。
上边代码一共有两个地方进行并发异常判断：

* 方法刚开始，这个时候先判断是否异常，没有的话继续
* 复制完当前当前列表的元素数组后，判断要找的元素的位置是否超了列表数组长度。因为上边i>=size这里已经可以确定要找的元素的下标在当前列表的长度范围内，而下边拿到数组后却发现下标超了，这说明在这两句话之间，列表元素减少了。**要注意的是：根据上述代码，这两句话之间若原列表添加了元素，那么是不会侦测到异常的，需要在下次调用next()的时候在checkForComodification()的时候侦测**

#### remove()方法
``` java
	public void remove() {
            if (lastRet < 0)//调用remove的时候，lastRet一定是大于等于0的
                throw new IllegalStateException();
            checkForComodification();//开头也像next()一样检查异常

            try {
                ArrayList.this.remove(lastRet);//对原列表进行remove
                cursor = lastRet;//游标回到上一个元素的位置
                lastRet = -1;//又变回-1
                expectedModCount = modCount;//这个很重要，因为调用原列表的remove方法后，modCount会-1，所以如果情况正常，expectedModCount和modCount的值必须相等的
            } catch (IndexOutOexpectedModCountfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
```
上述就是remove()方法的实现。首先，判断lastRet的值，不能小于0，为什么呢？有的人说这里初始的时候设置的-1，而且别的地方并没有对这个值进行设置，那么这里会报错啊？**因为调用remove()之前一定要先调用next()，调用next()方法后lastRet指向的是当前遍历的元素，所以这个时候lastRet一定大于等于0**.
上述代码接下来是调用ArrayList对象的remove()方法删除元素，调整游标和lastRet(变回-1)，更新expectedModCount(这一步很重要)，因为ArrayList的remove()方法修改列表结构后会更新modCount-1。

## ListIterator迭代器原理
上述的Itr迭代器是最常见的Iterator的实现，这个迭代器有如下两个缺陷：

* 只能往后遍历
* 每次迭代都得从当前列表的头部开始往后遍历

在ArrayList内部还定义了一个优化的迭代器类**ListIterator**，通过ArrayList对象的**listIterator(int index)**方法来获得，这个方法有个参数index表示游标初始定位到原列表的哪个位置，也就解决了上述第2个缺陷，看下具体的代码：
``` java
	public ListIterator<E> listIterator(int index) {
        if (index < 0 || index > size)
            throw new IndexOutOfBoundsException("Index: "+index);
        return new ListItr(index);
    }
    //ListItr继承自Itr内部类，实现了ListIterator接口，ListIterator是Iterator的子接口
    private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            super();
            cursor = index;
        }
        public boolean hasPrevious() {
            return cursor != 0;
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor - 1;
        }
    }
```
根据上述代码。ListItr继承自Itr内部类，实现了ListIterator接口，ListIterator是Iterator的子接口。**hasPrevious()**方法类似hasNext()方法，不过这里是判断前面是否还有元素；**nextIndex()**不要和next()方法搞混，这里只是返回下一个元素的下标，也就是cursor；**previousIndex()**和nextIndex()恰恰相反，返回的是上一个元素的下标。

#### previous()方法
``` java
	public E previous() {
            checkForComodification();
            int i = cursor - 1;
            if (i < 0)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i;
            return (E) elementData[lastRet = i];
        }
```
上述代码和next()方法的唯一却别在于，这里取的元素是cursor的上一个位置的元素，而next()取的是cursor位置的元素。

#### set()方法
``` java
	public void set(E e) {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.set(lastRet, e);//插入元素
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
```
ListIterator的特色，set()方法使得迭代器遍历的时候能对原来的列表进行更改元素的操作，而Itr里边只能进行remnove()。这个方法也很简单，照例先检查下lastRet，然后检查并发异常。主要的代码就一句，调用ArrayList的set()方法。

#### add()方法
``` java
	public void add(E e) {
            checkForComodification();

            try {
                int i = cursor;
                ArrayList.this.add(i, e);//在i位置处插入新的元素
                cursor = i + 1;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
```
上述代码是add()操作，**插入的位置是cursor**，然后cursor往后一位。

## 后记
Iterator还有其他的继承接口，PrimitiveIterator接口，这个接口重点是里边的PrimitiveIterator.OfDouble, PrimitiveIterator.OfInt, PrimitiveIterator.OfLong这几个接口，分别是针对不同的基本类型来迭代。具体的实现这里就不讲了，比较少用。
