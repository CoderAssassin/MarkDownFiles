## Java内存区域
[TOC]
Java虚拟机运行时管理的内存区域分为如下几个区域。
![javaMemory](https://github.com/CoderAssassin/markdownImg/blob/master/JavaVM/JavaMemory.png?raw=true)

#### 程序计数器
* 作用：记录当前线程执行的字节码的行号。
* 线程私有：每个时刻，一个处理器（对于多核处理器来说是一个内核）都只会执行一条线程中的指令，每条线程都需要一个独立的程序计数器，各条线程之间计数器互不影响，独立存储。
* 唯一一个没有OutOfMemoryError的内存区域。

#### 虚拟机栈
* 作用：描述方法执行时的内存模型，同样也是**线程私有**。
* 基本单位：栈帧，每个栈帧包含有**局部变量表、操作数栈、动态连接、返回地址等**
	* 局部变量表：存放编译器直到的**基本变量类型、直接引用和returnAddress类型**。**注意**：64位long和double占用2个局部变量空间。
* 异常：StackOverflowError和OutOfMemoryError。

#### 本地方法栈
* 作用：不同于虚拟机栈，本地方法栈是为Native方法(c++等语言的代码)服务的，线程私有。
* 异常：StackOverflowError和OutOfMemoryError。

#### 堆
* 作用：存储对象的实例，所有对象和数组都要在堆上分配内存。
* 异常：OutOfMemoryError。
* 是垃圾回收的主要区域。
* 线程共享。
* 内存分为新生代和老年代，新生代一般分为1个Eden区域和2个Survivor区域。
* 对于多线程的区域，一般会为每个线程划分出线程私有的缓冲区Thread Local Allocation Buffer，TLAB。线程会先用该缓冲区存，不够的时候再重新分配。

#### 方法区
* 作用：存放编译期生成的类信息、静态变量、常量、即时编译器编译后的代码等。
* 线程共享。
* 异常：OutOfMemoryError。
* 运行时常量池：存放编译期生成的字面量和符号引用，加载类后将类的这些信息放入常量池。

## HotSpot虚拟机对象
#### 对象的创建
1. 遇到new指令时，检查该指令的参数能否在方法区的运行池常量池中找到对应的类的符号引用，如果找不到说明该类还没有加载，那么需要加载该类。
2. 加载后，为对象分配内存，若内存都是整齐地存在一边，与空闲区域的交界处有一个指针，那么通过将该指针往空闲区域移动一定空间来分配内存，即“指针碰撞”；若不是整齐的，虚拟机会维护一个空闲区域列表，选择一块空闲区域分配，即“空闲列表”法。
3. 分配后，初始化内存空间为0(除对象头)。
4. 设置对象的所属类、类的元数据信息地址、对象哈希码以及GC分代年龄、是否启用偏向锁等。
5. 	new指令完成后，执行`<init>`方法，按代码所写进行初始化。

#### 对象的内存布局
* 对象头(MarkWord)。
	* 第一部分：存储运行时数据，GC分代年龄、哈希码、锁状态标志等。
	* 第二部分：类型指针。指向类元数据。

![neicunbuju](https://github.com/CoderAssassin/markdownImg/blob/master/JavaVM/MarkWord.png?raw=true)
例如32位虚拟机，25位是哈希码，4位是分代年龄，2位是锁状态标志，1位填充0.

* 实例数据。
* 对齐填充。因为地址要求8字节的整数倍，所以对象大小必须是8字节的整数倍。

> 分配时线程安全解决方法：一种是使用CAS原子操作加失败重试来保证更新操作的原子性；另一种是每个线程现在自身的TLAB分配内存，不够的时候再加锁分配新的内存。

#### 对象的访问定位
使用Reference指向对象，主流有句柄和直接指针方式。

* 句柄。在Java堆中划分一块句柄池区域，句柄池中包含到对象实例数据的指针。
![jubing](https://github.com/CoderAssassin/markdownImg/blob/master/JavaVM/jubing.png?raw=true)
* 直接指针：Reference直接存放到对象的指针。
![zhijiezhizhen](https://github.com/CoderAssassin/markdownImg/blob/master/JavaVM/zhijieyinyong.png?raw=true)

