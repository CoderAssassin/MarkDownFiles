[TOC]
看了不少关于synchronized的博客，还是准备做个总结，主要参考csdn zejian大神的博客来梳理一下，文末会附上文章出处引用。
## 什么是synchronized？
synchronized是java中实现线程同步的一个关键字。在多线程环境中，如果多个线程同时对共享数据进行修改的话，会造成数据不一致的问题。对共享数据加上synchronized关键字后，就是对数据的加锁，保证同一时刻只能有一条线程对该数据进行操作，这样的话，对数据的修改就能按照一种有序的顺序进行。

## synchronized的可重入性
synchronized是可重入锁，也就是说，当我们对一个对象加锁之后，我们持有了该对象的monitor，该monitor的count从0变到了1，当我们再次调用该对象的方法的时候，不需要再次去做获得monitor的动作，而是直接增加count的值。当子类继承父类的时候，子类也可以通过可重入锁调用父类的同步方法。

## synchronized的用法

主要有如下三种用法：

* 修饰实例方法：synchronized关键字加在实例方法上，此时是对调用该方法的实例对象加锁。
* 修饰静态方法：synchronized关键字加在类的静态方法上，此时锁加在这个类上，加在该类的Class对象上。
	* **问题探讨：Java中Class对象是存放在哪里的？**
		* 在JDK 1.6中，是存放在方法区中的；在JDK 1.7和1.8中都是存放在堆中的。
* 修饰同步代码块：使用方式synchronized(对象/类)，此时锁是加在括号里边的对象或者类的Class对象上。

## synchronized的底层实现
#### 获取锁的底层实现
每个对象都有个monitor(监视器锁)，当某条线程获取monitor就表示该线程占有该对象的锁，别的对象无法再获取锁，这样就实现了线程的同步。synchronized是可重入的锁，因此可以多次获得某个对象的锁，monitor会有一个计数器，表示锁被获取的次数，获取几次也同样需要解锁几次。
synchronized获取锁在JVM虚拟机中都是通过两个字节码指令monitorenter和monitorexit来实现，分别表示进入同步代码块和退出同步代码块。而同步方法是通过调用指令读取运行时常量池中的ACC_SYNCHRONIZED标志来实现的，底层依旧是monitor的两条指令。
``` java
	public synchronized void Synchronized_method(){
        System.out.println("方法同步");
    }

    public void Synchronized_block(){
        synchronized (this){
            System.out.println("代码块同步");
        }
    }

    public static synchronized void Synchronized_static(){
        System.out.println("类方法同步");
    }
```
反编译后得到：
``` java
	public synchronized void Synchronized_method();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String 方法同步
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 11: 0
        line 12: 8

  public void Synchronized_block();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #5                  // String 代码块同步
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit
        20: aload_2
        21: athrow
        22: return
      Exception table:
         from    to  target type
             4    14    17   any
            17    20    17   any
      LineNumberTable:
        line 15: 0
        line 16: 4
        line 17: 12
        line 18: 22
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 17
          locals = [ class huawei/SynchronizedTest, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4

  public static synchronized void Synchronized_static();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #6                  // String 类方法同步
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 21: 0
        line 22: 8
```
通过观察上边代码可以发现，对于synchronized关键字加在方法上的情况，flags里多了个ACC_SYNCHRONIZED，以这个标志表示该方法是同步方法，然后底层依旧是需要获取对象的监视器锁，调用那两条指令；如果synchronized是按同步代码块的实现方式实现的话，这里就出现了monitorenter和monitorexit两条指令，分别表示进入同步代码块和退出同步代码块，后边读了一个monitorexit，jvm会自动产生一个异常处理器，用来处理所有的异常，当异常结束时会调用该处的monitorexit释放monitor；如果是加在静态方法上的话，和加在普通方法上一样，也有个ACC_SYNCHRONIZED标志。

#### Java对象头
JVM中，对象的组成有如下几个部分：
* 对象头(Mark Word)：第一部分存储对象的HashCode、GC分代年龄、锁状态信息等；第二部分存储对象的类型指针，指向它的类的元数据的指针。若对象是java数组，那么在对象中必须有一块用于记录数据长度的数据。
如下图表示对象头第一部分在不同锁情况下的存储信息分布：
![markword](https://github.com/CoderAssassin/markdownImg/blob/master/Java/markword.jpg?raw=true)
* 实例数据：存储对象的数据
* 对其填充：因为JVM规定对象的起始地址需要是8字节的整数倍，所以这里需要填充。

#### synchronized的优化
synchronized是一个重量级的锁，因为监视器锁monitor的底层使用的是操作系统的互斥所Mutex Lock实现，操作系统实现线程之间的切换需要从用户态转换到核心态，切换需要很长时间。JDK 1.6以后，引入了轻量级锁和偏向锁的概念之后，对synchronized进行了优化。
关于轻量级锁，重量级锁，自旋锁等，请参考另一篇博客：[java锁优化](https://coderassassin.github.io/2018/07/20/jvm-threadsafe/)，这里不再细讲。


## Reference
* [https://blog.csdn.net/javazejian/article/details/72828483](https://blog.csdn.net/javazejian/article/details/72828483)
* [https://blog.csdn.net/zhoufanyang_china/article/details/54601311](https://blog.csdn.net/zhoufanyang_china/article/details/54601311)