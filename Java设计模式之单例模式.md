[TOC]
## 单例模式的基本概念
单例模式的主要目的是保证再Java程序中，某个类的实例只有一个存在。
一般来说，单例类由静态对象，私有的构造函数和静态的创建实例的方法getInstance()工厂方法组成。

## 单例模式的特点
1. 单例类只能创建一个实例。
2. 单例实例只能由单例类自身来创建。
3. 单例类必须向全局其他类提供来单例实例。

## 单例模式和静态类的区别
1. 单例类可以继承和被继承，而静态方法不行。
2. 静态类的对象执行后会被释放，然后被hc清理，不会存在内存中。
3. 静态类在第一次运行的时候会初始化，单例可以延迟加载。
4. 单例模式创建的对象常驻内存，可以节约很多资源。
5. 静态方法访问效率很高。
6. 单例模式容易测试。

## 单例模式的实现
#### 懒汉模式
``` java
public class Singleton{
	private static Singleton instance;//私有静态未实现对象
    private Singleton(){}
    public static Singleton getInstance(){//公有方法创建对象实例
    	if(instance==null){//一次判空条件
        	instance=new Singleton();
        }
        return instance;
    }
}
```
懒汉模式类里边的对象为私有的未实现对象，对于对象的创建，通过调用类的公有静态方法来创建。懒汉模式就是类加载的时候并没有创建类对象，而是在需要的时候再创建，所以是**懒加载**的。但是这样的话并发安全性很低，当多条线程并发创建新的实例时，会将前面的实例覆盖掉。

#### 线程安全的懒汉模式
``` java
public class Singleton{
	private static Singleton instance;
    private Singleton(){}
    public static Synchronized Singleton getInstance(){//加同步锁
    	if(instance==null){
        	instance=new Singleton();
        }
        return instance;
    }
}
```
和线程不安全的懒汉模式唯一的区别是创建实例的方法加了个同步锁Synchronized，锁对象是SingletonDemo类，这样的话只能有一条线程创建实例对象。

#### 饿汉模式
``` java
public class Singleton{
	private static Singleton instance=new Singleton();
    private Singleton(){}
    public static Singleton getInstance(){
    	return instance;
    }
}
```
饿汉模式和前面的懒汉模式的区别是，饿汉模式在加载类的时候就已经创建了一个实例对象。

#### 静态内部类加载
``` java
public class Singleton{
	private static class SingletonHolder{
    	private static final Singleton INSTANCE=new Singleton();//内部类里创建外部类实例
    }
    private Singleton(){}
    public static final Singleton getInstance(){
    	return SingletonHolder.INSTANCE;//返回内部类的静态字段，第一次调用该方法的时候会初始化内部类
    }
}
```
这种方式将外部类对象的创建放在静态内部类的静态字段里，根据静态内部类的特点，只有在使用静态内部类的时候才会进行初始化工作，因此可以实现懒加载。

#### 枚举方式
``` java
public enum Singleton{
	INSTANCE;
    public void method(){
    	doSomething;
    }
}
```
Effective Java作者提倡的一种方式，不仅避免多线程同步问题，而且防止多次创建对象。

#### 双重校验锁
``` java
public class Singleton{
	private volatile static Singleton singleton;//volatile标记对象
    private Singleton(){}
    public static Singleton getSingleton(){
    	if(singleton==null){//第一次判断
        	Synchronized(Singleton.class){//加锁
            	if(singleton==null){//第二次判断
                	singleton=new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
这种方式一共两次判断对象是否为空，当为空的时候，通过加锁使得每次只有一条线程进行对象的初始化，此时还需要进行一次判空，这是为了防止线程A拿到锁实例化以后，线程B又拿到锁继而想当然地自个儿再实例化，保证了单例特点；并且对象是volatile修饰的，能立马被其他挡在第一次判断前的线程看到。这种方法既保证了单例又保证了线程安全。


## Reference
* [1.Java单例模式——并非看起来那么简单](https://blog.csdn.net/goodlixueyong/article/details/51935526)
* [2.java设计模式之 单例模式](https://www.cnblogs.com/kuoAT/p/6725808.html)
