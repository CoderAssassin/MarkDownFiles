[TOC]
平时在写java代码的时候，当需要用到对一个列表进行排序，我们习惯性地调用用Collections.sort()来进行排序，但是我们对这个方法的底层排序原理一无所知，所以，今天让我们来看一下这个方法的底层的实现原理，揭开它的神秘面纱(所有代码基于jdk 1.8)。
首先让我们来看一下在Collections工具类里边的入口函数Collections.sort()：
``` java
	//按照升序对list的元素进行排序，所有的元素必须是实现Comparable接口，所有的元素之间可以进行比较而不会抛出ClassCastException；排序的方法是稳定的
	public static <T extends Comparable<? super T>> void sort(List<T> list) {
        list.sort(null);
    }
    //传入自定义的比较器进行排序
    public static <T> void sort(List<T> list, Comparator<? super T> c) {
        list.sort(c);
    }
```
上述是两个入口方法，在第一个方法里，传入的list的元素必须是实现了Comparable接口的，第二个方法的元素是任何类型，但是需要传入Comparator对象来设置比较规则。两个方法都是**稳定的排序**，并且最终都是调用list对象的sort()方法，进入该方法，该方法是List接口的默认方法(jdk 1.8允许在接口中定义默认的方法)：
``` java
	default void sort(Comparator<? super E> c) {
        Object[] a = this.toArray();
        Arrays.sort(a, (Comparator) c);
        ListIterator<E> i = this.listIterator();
        for (Object e : a) {//迭代地将排好序的对象更换到原数组
            i.next();
            i.set((E) e);
        }
    }
```
上述方法先复制一份元素到数组a中，**然后调用Arrays.sort()进行排序**，最后将排好序的元素迭代地添加到当前列表中。所以，本质上还是调用的Arrays.sort()进行排序，那么正好来看一下这个方法是怎么个实现，进入Arrays工具类：
``` java
	public static <T> void sort(T[] a, Comparator<? super T> c) {
        if (c == null) {//没有设定排序规则，用默认排序
            sort(a);
        } else {
            if (LegacyMergeSort.userRequested)
                legacyMergeSort(a, c);
            else
                TimSort.sort(a, 0, a.length, c, null, 0, 0);
        }
    }
```
上述代码先是判断是否传入了Comparator对象，若是的话直接调用默认sort()；否则的话，判断是否使用旧的归并排序，是的话调用legacyMergeSort()；否则调用TimSort.sort()，这个排序算法是来自1993年的一个叫Peter McIlroy的论文。
下面首先看下sort()方法：
``` java
	public static void sort(Object[] a) {
        if (LegacyMergeSort.userRequested)//调用老归并方法
            legacyMergeSort(a);
        else
            ComparableTimSort.sort(a, 0, a.length, null, 0, 0);
    }
    //老归并方法(在未来会被移除)
    private static void legacyMergeSort(Object[] a) {
        Object[] aux = a.clone();//克隆一份
        mergeSort(aux, a, 0, a.length, 0);//调用归并排序(该方法将来会被移除)
    }
    //归并排序，虽然会被移除，用作平时代码参考也可以
    private static void mergeSort(Object[] src,
                                  Object[] dest,
                                  int low,
                                  int high,
                                  int off) {
        int length = high - low;

        // 长度小于7用插入排序
        if (length < INSERTIONSORT_THRESHOLD) {
            for (int i=low; i<high; i++)
                for (int j=i; j>low &&
                         ((Comparable) dest[j-1]).compareTo(dest[j])>0; j--)
                    swap(dest, j, j-1);
            return;
        }

        // 调整排序好后存到目标数组的下标范围
        int destLow  = low;
        int destHigh = high;
        low  += off;
        high += off;
        int mid = (low + high) >>> 1;
        mergeSort(dest, src, low, mid, -off);//递归调用
        mergeSort(dest, src, mid, high, -off);

        // 如果当前左右两个列表已经排好序了，那么直接复制到目的数组，然后返回
        if (((Comparable)src[mid-1]).compareTo(src[mid]) <= 0) {
            System.arraycopy(src, low, dest, destLow, length);
            return;
        }

        // 将排好序的部分复制过去
        for(int i = destLow, p = low, q = mid; i < destHigh; i++) {
            if (q >= high || p < mid && ((Comparable)src[p]).compareTo(src[q])<=0)
                dest[i] = src[p++];
            else
                dest[i] = src[q++];
        }
    }
```
上述代码会根据系统设置选择调用归并排序还是调用ComparableTimSort.sort()，归并排序可能在将来的某个版本中被移除，但是可以作为模板代码参考，那么我们接下来看ComparableTimSort里的sort()方法是怎么实现的：
``` java
	static void sort(Object[] a, int lo, int hi, Object[] work, int workBase, int workLen) {
        assert a != null && lo >= 0 && lo <= hi && hi <= a.length;

        int nRemaining  = hi - lo;//长度
        if (nRemaining < 2)//长度为0或者1不用排序
            return;

        // 如果长度小于32，那么调用使用mini-TimSort算法进行排序，而不是用的归并
        if (nRemaining < MIN_MERGE) {
            int initRunLen = countRunAndMakeAscending(a, lo, hi);//计算从lo开始的连续的升序或者降序序列的长度
            binarySort(a, lo, hi, lo + initRunLen);
            return;
        }

        ComparableTimSort ts = new ComparableTimSort(a, work, workBase, workLen);
        int minRun = minRunLength(nRemaining);//计算得到对于当前数组长度来说的短序列的长度限界
        do {
            // 连续的升序或降序的元素长度
            int runLen = countRunAndMakeAscending(a, lo, hi);

            // 连续的长度短语最小长度限界，那么使用binarySort()
            if (runLen < minRun) {
                int force = nRemaining <= minRun ? nRemaining : minRun;//最多对minRun长度的元素使用binarySort()排序
                binarySort(a, lo, lo + force, lo + runLen);
                runLen = force;
            }

            // 将当前次的状态记录，可能的话会进行合并
            ts.pushRun(lo, runLen);
            ts.mergeCollapse();

            // 继续下一个run循环
            lo += runLen;
            nRemaining -= runLen;
        } while (nRemaining != 0);

        // 最后对所有剩余的runs进行合并
        assert lo == hi;
        ts.mergeForceCollapse();
        assert ts.stackSize == 1;
    }
    //返回最长的升序序列或者降序序列的长度
    private static int countRunAndMakeAscending(Object[] a, int lo, int hi) {
        assert lo < hi;
        int runHi = lo + 1;
        if (runHi == hi)
            return 1;

        // 找到连续升序或者降序的终点
        if (((Comparable) a[runHi++]).compareTo(a[lo]) < 0) { // 降序
            while (runHi < hi && ((Comparable) a[runHi]).compareTo(a[runHi - 1]) < 0)
                runHi++;
            reverseRange(a, lo, runHi);
        } else {                              // 升序
            while (runHi < hi && ((Comparable) a[runHi]).compareTo(a[runHi - 1]) >= 0)
                runHi++;
        }

        return runHi - lo;
    }
    //将降序数组的连续的序列反转
    private static void reverseRange(Object[] a, int lo, int hi) {
        hi--;
        while (lo < hi) {
            Object t = a[lo];
            a[lo++] = a[hi];
            a[hi--] = t;
        }
    }
    //每次将n除以2，得到小于等于n最大的数，然后+1
    private static int minRunLength(int n) {
        assert n >= 0;
        int r = 0;      // 这个r很妙，如果当前右移的位是1的话，只要右移的位有1的话，就一直是1
        while (n >= MIN_MERGE) {
            r |= (n & 1);
            n >>= 1;
        }
        return n + r;
    }
```
上述代码中，对于长度小于32的数组，使用的是binarySort()来排序；否则的话，接下来的排序思想是，每次我们都根据当前的剩余待排序元素的个数，先确定一个minRun，这个变量的意思是：比32小的最大的2的幂次数(可能会+1)，比如当前剩余元素个数是33，那么最后返回的是17，如果是32，那么返回的是16。确定了这个数后，那么就开始进入循环，首先计算从当前lo位置开始的最长的连续升序或者降序序列的长度runLen，若该长度小于32，那么选取min(minRun, nRemaining)的数调用binarySort()进行排序。当每次循环完成后，会记录本次循环的信息，并且会将数组进行合并。
总结一下：当数组长度小于32会直接调用binarySort()排序，当数组长度大于等于32，这里会对原数组分好几趟(或者叫好几次run)分别进行排序，每次根据最大连续升序(降序)的序列长度与minRun的关系进行排序，然后合并当前排好序的数组。
那么binarySort()到底是怎么实现的呢？我们来看看：
``` java
	//这个start是[lo,hi)范围内第一个不在升序或者降序序列的数的下标
	private static void binarySort(Object[] a, int lo, int hi, int start) {
        assert lo <= start && start <= hi;
        if (start == lo)
            start++;
        for ( ; start < hi; start++) {
            Comparable pivot = (Comparable) a[start];//获得比较器

            int left = lo;
            int right = start;
            assert left <= right;
            //pivot下标的数要不比左边所有的数大(降序)，要不都小(升序)
             //循环遍历，以升序排序为例，这里是用二叉搜索找到pivot的数在left到right之间的插入位置
            while (left < right) {
                int mid = (left + right) >>> 1;
                if (pivot.compareTo(a[mid]) < 0)
                    right = mid;
                else
                    left = mid + 1;
            }
            assert left == right;

            int n = start - left;
            //用switch是一个小优化，元素个数大于等于3以后才会调用System.arraycopy()
            switch (n) {
                case 2:  a[left + 2] = a[left + 1];
                case 1:  a[left + 1] = a[left];
                         break;
                default: System.arraycopy(a, left, a, left + 1, n);
            }
            a[left] = pivot;
        }
    }
```
上边就是binarySort()的实现，整体算法思路是采用的二叉搜索来找到每个无序的数在前边有序的序列里的插入位置，然后将元素后移一位插入，算是一种优化后的插入排序。
最后，最上边的Arrays类里边的sort()方法里最后还有个TimSort.sort()调用，这个和ComparableTimSort.sort()方法基本一致每一的差别是这里会传入自定义的Comparator对象参数设定比较规则。

## 小结
至此，Collections.sort()的排序原理算是清楚了，总的来说，调用的是Arrays里的sort，这个sort总的根据是否用老版的归并排序还是新版的TimSort来分，老版的归并排序快要被淘汰了。这里的TimSort比较陌生，最终都是调用的一个binarySort()方法，也是整个排序的核心，总体思想是：采用二叉搜索的方法找到的待排序元素在已排序序列中的插入位置，然后插入。