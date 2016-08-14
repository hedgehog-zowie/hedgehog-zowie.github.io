---
title: Object类详解
date: 2016-04-14 15:18:01
toc: true
tags:
- Java
categories:
- Java基础

---

Object类是Java中所有类的祖先，在Java中每个类都是由它扩展而来的。如果没有明确地指出超类，Object类就被认为是这个类的超类。
可以使用Object类型的变量引用任何类型的对象：
`Object obj = new MyObject();`
但是，要想对其中的内容进行具体的操作，还需要清楚对象的原始类型，并进行相应的转化：
`MyObject mObj = (MyObject) obj;`
在Java中只有基本类型不是对象，例如：数值、字符和布尔型的值都不是对象。所有的数组类型，不管是对象数组还是基本类型数组都扩展于Object类。

# 方法
## public Object();
Java中规定：在类定义过程中，对于未定义构造函数的类，默认会有一个无参数的构造函数，作为所有类的祖先，Object类自然要反映出此特性，在源代码中，未给出Object类构造函数定义，但实际上，此构造函数是存存的。

## private static native void registerNatives()；
将本地函数向VM进行登记，将C/C++中的方法映射到Java中的native方法，实现方法命名的解耦。
此方法由其后紧跟的静态代码调用：
```
    static {
        registerNatives();
    }
```

## public final native Class<?> getClass()；
getClass()是一个native方法，返回这个对象的运行时类对象，效果与Object.class相同。
首先解释下"类对象"的概念：在Java中，类是是对具有一组相同特征或行为的实例的抽象并进行描述，对象则是此类所描述的特征或行为的具体实例。作为概念层次的类，其本身也具有某些共同的特性，如都具有类名称、由类加载器去加载，都具有包，具有父类，属性和方法等。于是，Java中有专门定义了一个类，Class，去描述其他类所具有的这些特性，因此，从此角度去看，类本身也都是属于Class类的对象。为与经常意义上的对象相区分，在此称之为"类对象"。

## public native int hashCode()
hashCode()是一个native方法，返回一个整形数值，表示该对象的哈希码值。
hashCode()具有如下约定：
1. 在Java应用程序执行期间，对于同一个对象多次调用hashCode()方法时，其返回的哈希码是相同的，前提是将对象进行equals比较时所用的信息未做修改。在某一应用程序的一次执行到同一应用程序的另一次执行，该整形数值无需保持一致。
2. 如果根据equals（object）方法，两个对象是相等的，则hashcode必须相同。
3. 如果两个对象的hashcode相同，这两个对象不一定相等。

`需要注意的是：为不相等的对象生成不同的hashcode可以提高哈希表的性能。`

## public boolean equals(Object obj)
equals()方法判断两个对象是否相等，在Object类中，equals表示的是同一个对象。
代码如下：

```
    public boolean equals(Object obj) {
        return (this == obj);
    }
```
`注意：重写了equals方法后需要重写hashCode方法。`

## protected native Object clone() throws CloneNotSupportedException
clone()是一个native方法，其作用是创建并返回此对象的副本，返回的是一个引用，指向的是新clone出来的对象，此对象与原对象分别占用不同的堆空间。此方法执行的是对象的浅拷贝而不是深拷贝。

`
注意：
1. 不可在子类中创建Object对象，并直接clone，原因如下：clone()方法是protected的，protected关键字的含义是：在同一个包内或者不同包的子类可以访问，当两个类不在同一个包中的时候，在子类内部且子类的引用才可以访问父类的protected成员；在子类内部，父类的引用并不能访问protected成员。
例：

```
Object obj = new Object();
Object cloneObj = obj.clone();
```
报错为："The method clone() from the type Object is not visible"
2. 使用clone()方法的子类需要实现Cloneable接口，否则会抛出异常：java.lang.CloneNotSupportedException
例：
```
public class CloneTest implements Cloneable{

    public static void main(String args[]) throws CloneNotSupportedException {
        CloneTest obj = new CloneTest();
        CloneTest cloneObj = (CloneTest) obj.clone();
    }

}
```
`

## public String toString()
toString()方法返回该对象的字符串表示。代码如下：

```
    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }
```

## public final native void notify()
唤醒在此对象监视器上等待的单个线程。如果所有线程都在此对象上等待，则会选择唤醒其中一个线程。选择是任意性的，并在对实现做出决定时发生。线程通过调用其中一个wait方法，在对象的监视器上等待。

## public final native void notifyAll()
唤醒在此对象监视器上等待的所有线程。线程通过调用其中一个wait方法，在对象的监视器上等待。

## public final native void wait(long timeout) throws InterruptedException
在其他线程调用此对象的notify()方法或notifyAll()方法，或者超过指定的时间量前，导致当前线程等待。
`timeout单位为毫秒。`

## public final void wait(long timeout, int nanos) throws InterruptedException
在其他线程调用此对象的notify()方法或notifyAll()方法，或者其他某个线程中断当前线程，或者已超过某个实际时间量前，导致当前线程等待。
此方法类似于一个参数的wait方法，但它允许更好地控制 在放弃之前等待通过的时间量。用毫微秒度量的实际时间量可以通过以下公式计算出来：
`1000000 * timeout + nanos`
在其他所有方面，此方法执行的操作与带有一个参数的wait(long)方法相同，需要特别指出的是，wait(0,0)与wait(0)相同。

## 简单的生产消费模型
一段简单的存钱/取钱代码，用来体现一下wait/notify的作用。

```

    static class Account {
        private long balance;

        public synchronized void take(long money) throws InterruptedException {
            while (balance < money) {
                System.out.println("余额不足, 需取: " + money + "; 余额: " + balance);
                wait();
            }
            balance -= money;
            System.out.println("取出: " + money + "; 余额: " + balance);
        }

        public synchronized void put(long money) throws InterruptedException {
            balance += money;
            System.out.println("存入: " + money + "; 余额: " + balance);
            notifyAll();
        }
    }

    static class TakeThread extends Thread {
        private Account account;

        public TakeThread(Account account) {
            this.account = account;
        }

        public void run() {
            Random random = new Random();
            while (true) {
                int money = random.nextInt(100);
                try {
                    account.take(money);
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    static class PutThread extends Thread {
        private Account account;

        public PutThread(Account account) {
            this.account = account;
        }

        public void run() {
            Random random = new Random();
            while (true) {
                int money = random.nextInt(100);
                try {
                    account.put(money);
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
    
```

## public final void wait() throws InterruptedException
= wait(0)

## protected void finalize() throws Throwable
当垃圾回收器确定不存在对该对象的更多引用时，由对象的垃圾加收器调用此方法。子类重写finalize方法，以配置系统资源或执行其他清除。

# 参考资料
[Java总结篇系列：java.lang.Object](http://www.cnblogs.com/lwbqqyumidi/p/3693015.html)

jdk api

