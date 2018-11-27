---
title: Java中单例模式的几种实现
date: 2018-06-03 22:43:33
tags:
---
在Java中单例模式有几种实现，在不同的使用场景下可以选择适当的实现。下面就让我们来看下具体的实现和各自的优缺点。
	
## 饿汉模式

```Java
public class HungrySingleton {

    private HungrySingleton() {
        System.out.println("HungrySingleton is created");
    }

    private static final HungrySingleton HUNGRY_SINGLETON = new HungrySingleton();

    public static HungrySingleton getInstance() {
        return HUNGRY_SINGLETON;
    }
}
```

这种方式中实例被 *static* 和 *final* 修饰，在类加载到内存中时就会被初始化，不存在线程安全问题。
缺点是无法实现延迟加载，同时在实例构造时需要依赖参数或配置文件的场景下也无法使用。

## 懒汉模式

```Java
public class LazySingleton {

    private LazySingleton() {
        System.out.println("LazySingleton is created");
    }

    private static LazySingleton lazySingleton;

    public static LazySingleton getInstance() {
        if (lazySingleton == null) {
            lazySingleton = new LazySingleton();
        }
        return lazySingleton;
    }
}
```

下面我们再来看一下懒汉模式，这种方式虽然实现了延迟加载，但是在多线程场景下无法使用，原因是在多线程情况下当多个线程同时进入 *lazySingleton == null* 判断分支中，就会创建多个实例。

## 双重检查模式

在懒汉模式的基础上，我们自然而然会想到使用加锁的方式控制多个线程进入判断中，但是单例模式是一种“一次写多次读”的场景，如果在 *if* 判断前直接加锁，将会严重降低单例获取的性能.
因此，双重检查模式使用了一次前置判断，当实例不存在时才进行加锁操作，同时在加锁的同步代码块中也进行了一次判断，防止多个实例进入锁等待后创建多个实例。

```Java
public class DoubleCheckSingleton {

    private DoubleCheckSingleton() {
        System.out.println("DoubleCheckSingleton is created");
    }

    private static DoubleCheckSingleton doubleCheckSingleton;

    public static DoubleCheckSingleton getInstance() {
        if (doubleCheckSingleton == null) {
            synchronized (DoubleCheckSingleton.class) {
                if (doubleCheckSingleton == null) {
                    doubleCheckSingleton = new DoubleCheckSingleton();
                }
            }
        }
        return doubleCheckSingleton;
    }
}
```

上面的代码实现会有什么问题吗？如果你够细心的话你就会发现 *doubleCheckSingleton = new DoubleCheckSingleton()* 这个操作是一个非原子操作，在JVM中把这句代码分了3步：

1. 为 *doubleCheckSingleton* 分配内存空间
2. 执行 *DoubleCheckSingleton* 的构造函数进行初始化操作
3. 将 *doubleCheckSingleton* 指向分配的内存空间（这步执行完后 *doubleCheckSingleton* 才不为*null* ）


由于JVM即时编译器会进行指令重排，因此最后的执行顺序可能是 *1-2-3* 或 *1-3-2* ，当执行顺序是 *1-3-2* ，如果线程一执行完3后还未执行2，此时有线程二抢占CPU，拿到不为 *null* 的 *doubleCheckSingleton* 实例进行使用，但是实例并未被初始化，因此使用报错。
为了解决这个问题，像 *[The "Double-Checked Locking is Broken" Declaration](https://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)* 中提到的，我们需要使用 *volatile* 修饰实例变量 *doubleCheckSingleton* ，这样就可以禁止JVM即时编译器对 *doubleCheckSingleton* 实例对象的操作进行执行重排。需要注意 *volatile* 的这种使用只在 *JDK1.5* 及以上有效。让我们看一下最终的双重检查模式实现。

```Java
public class DoubleCheckSingleton {

    private DoubleCheckSingleton() {
        System.out.println("DoubleCheckSingleton is created");
    }

    private volatile static DoubleCheckSingleton doubleCheckSingleton;

    public static DoubleCheckSingleton getInstance() {
        if (doubleCheckSingleton == null) {
            synchronized (DoubleCheckSingleton.class) {
                if (doubleCheckSingleton == null) {
                    doubleCheckSingleton = new DoubleCheckSingleton();
                }
            }
        }
        return doubleCheckSingleton;
    }
}
```

## 静态内部类模式

```Java
public class InnerClassSingleton {

    private InnerClassSingleton() {
        System.out.println("InnerClassSingleton is created");
    }

    private static class SingletonHolder {
        private static final InnerClassSingleton INSTANCE = new InnerClassSingleton();
    }

    public static InnerClassSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

私有的静态内部类保证了只有 *getInstance()* 才能访问到实例，并由JVM保证了创建实例时的线程安全问题，同时获取实例的时候不需要进行加锁同步操作，也不依赖JDK版本。这是《*Effective Java*》第一版中推荐的写法。

## 枚举模式

```Java
public enum  EnumSingleton {
    INSTANCE;

    EnumSingleton() {
        System.out.println("EnumSingleton is created");
    }

    public static void main(String[] args) {
        EnumSingleton enumSingleton1 = EnumSingleton.INSTANCE;
        EnumSingleton enumSingleton2 = EnumSingleton.INSTANCE;
        System.out.println("equals: " + (enumSingleton1 == enumSingleton2));
    }
}
```

枚举实例的创建是线程安全的，枚举实例获取也不需要加锁同步，同时还防止了其他版本的单例实现中可以通过反射调用单例的私有构造方法，但是枚举中的其他方法的线程安全需要实现方自己负责。这是《*Effective Java*》第二版中推荐的写法。

## 总结

以上主要介绍了Java中单例的5种实现方式，分别是饿汉模式、懒汉模式、双重检查模式、静态内部类模式、枚举模式，实际使用中还是要考虑使用的场景，是否需要延迟加载、是否需要序列化等。

#### 参考文献

* [深入浅出单实例SINGLETON设计模式](https://coolshell.cn/articles/265.html)
* [The "Double-Checked Locking is Broken" Declaration](https://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)