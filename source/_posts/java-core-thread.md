---
title: 多线程总结
date: 2016-04-19 15:48:26
toc: true
tags:
- Java
- 多线程
categories:
- Java基础

---

线程是程序执行流的最小单元，多线程程序在较低层次上扩展了多任务的概念：一个程序同时执行多个任务。通常每一个任务称为一个线程(thread)，它是线程控制的简称。可以同时运行一个以上线程的程序称为多线程程序(multithreaded)。

# 线程的状态

Java的线程如下有6种状态：
NEW（新建）
RUNNABLE（就绪，可运行）
BLOCKED（被阻塞）
WAITING（等待）
TIMED_WAITING（计时等待）
TERMINATED（被终止）

可以使用Thread.getState()方法获取线程的当前状态。在Thread类中线程状态由一个整型变量threadStatus表示，getState()方法调用VM类的toThreadState()方法，根据threadStatus某个位置上是否为1来判断线程状态，代码如下：

```
(var0 & 4) != 0?State.RUNNABLE:((var0 & 1024) != 0?State.BLOCKED:((var0 & 16) != 0?State.WAITING:((var0 & 32) != 0?State.TIMED_WAITING:((var0 & 2) != 0?State.TERMINATED:((var0 & 1) == 0?State.NEW:State.RUNNABLE)))));
```

## NEW

当用new操作符创建一个新线程时，如new Thread()，该线程还没有运行，此时线程状态是NEW。

## RUNNABLE

一旦调用start()方法，线程处于RUNNABLE状态。
`一个RUNNABLE状态的线程可能正在运行也可能没有运行，这取决于操作系统给线程提供运行的时间`。

## BLOCKED

当一个线程试图获取一个内部对象锁，而该锁被其他线程持有，则该线程进入BLOCKED状态。

## WAITING

当一个线程等待另一个线程通知调度器一个条件时，进入WAITING状态，如调用：Object.wait(), Thread.join(), Lock.lock(), Condition.await()时。

## TIMED_WAITING

有几个方法有超时参数，调用它们将导致线程进入TIMED_WAITING状态，如调用：Thread.sleep(long millis), Object.wait(long timeout), Lock.tryLock(), Condition.await(long time, TimeUnit unit)时。

## TERMINATED

* 因为run方法正常退出而自然终止。
* 因为一个没有捕获的异常而终止。

`stop()、suspend()、resume()已过时，不要使用。`

# 线程属性

线程属性包括：线程优先级、守护线程、线程组以及处理未捕获异常的处理器。

## 线程优先级

在Java中每一个线程都有一个优先级，使用setPriority()方法可以设置线程的优先级，可以将优先级置为在MIN_PRIORITY(在Thread类中定义为0)和MAX_PRIORITY（在Thread类中定义为10）之间的任何值。NORM_PRIORITY被定义为5。

`注意：线程优先级高度依赖于系统，Windows有7个优先级，Sun为Linux提供的Java虚拟机，线程的优先级被忽略——所有的线程具有相同的优先级。`

## 守护线程

通过调用方法setDaemon(true)来将线程设置为守护线程，守护线程的唯一作用是为其他线程提供服务。当只剩下守护线程时，虚拟机就退出了，所以守护线程中永远不应该去访问固有资源，如文件、数据库，因为它会在任何时候甚至在一个操作的中间发生中断。

## 未捕获异常处理器

该处理器必须实现Thead.UncaughtExceptionHandler接口，该接口只有一个方法：void uncaughtException(Thread t, Throwable e)。
使用setUncaughtExceptionHandler()方法为线程安装一个处理器，也可以使用静态方法Thread.setUncaughtExceptionHandler()为所有线程安装一个默认的处理器。

# 线程的创建

创建线程有三种方法：
* 继承Thread类
* 实现Runnable接口
* 实现Callable接口

## 继承Thread类

步骤如下
1. 继承Thread类，重写run方法
2. 创建线程对象
3. 执行start方法

```
public class ExtendsThread extends Thread{
    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            System.out.println(getName());
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String args[]) {
        ExtendsThread subThread1 = new ExtendsThread();
        subThread1.setName("subThread1");
        ExtendsThread subThread2 = new ExtendsThread();
        subThread2.setName("subThread2");
        subThread1.start();
        subThread2.start();
    }
}
```

输出结果(不固定)：

```
subThread1
subThread2
subThread2
subThread1
subThread2
subThread1
subThread2
subThread1
subThread2
subThread1
subThread2
subThread1
subThread2
subThread1
subThread1
subThread2
subThread1
subThread2
subThread1
subThread2
```

`注意：不要调用Thread类或Runnable对象的run方法，直接调用run方法，只会执行同一个线程中的任务，而不会启动新线程。应该调用Thread.start方法，这个方法将创建一个执行run方法的新线程。`

## 实现Runnable接口

1. 创建Runnable的实现类，重写run方法
2. 创建该Runnable实现类的对象，并以该对象为参数创建Thread实例
3. 执行start方法

```
public class ImplRunnable {
    public static void main(String args[]){
        MyRunnable myRunnable = new MyRunnable();
        Thread subThread1 = new Thread(myRunnable, "subThread1");
        Thread subThread2 = new Thread(myRunnable, "subThread2");
        subThread1.start();
        subThread2.start();
    }
    static class MyRunnable implements Runnable{
        @Override
        public void run() {
            for (int i = 0; i < 10; i++) {
                System.out.println(Thread.currentThread().getName());
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

输出结果（不固定）：

```
subThread1
subThread2
subThread1
subThread2
subThread1
subThread2
subThread1
subThread2
subThread1
subThread2
subThread2
subThread1
subThread1
subThread2
subThread1
subThread2
subThread2
subThread1
subThread2
subThread1
```

## 实现Callable接口

1. 创建Callable的实现类，重写call()方法
2. 创建该Callable实现类的对象，并以该对象为参数创建FutureTask对象
3. 以该FutureTask对象为参数，创建Thread实例
4. 执行start方法，并可调用FutureTask对象的方法获取线程执行的状态及返回结果。

```
public class ImplCallable {
    
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        MyCallable myCallable1 = new MyCallable(10);
        MyCallable myCallable2 = new MyCallable(20);
        FutureTask<Integer> futureTask1 = new FutureTask(myCallable1);
        FutureTask<Integer> futureTask2 = new FutureTask(myCallable2);
        Thread subThread1 = new Thread(futureTask1);
        Thread subThread2 = new Thread(futureTask2);
        subThread1.start();
        subThread2.start();
        System.out.println(futureTask1.get());
        System.out.println(futureTask2.get());
    }

    static class MyCallable implements Callable<Integer>{
        private int num;

        public MyCallable(int num) {
            this.num = num;
        }

        public int getNum() {
            return num;
        }

        public void setNum(int num) {
            this.num = num;
        }

        @Override
        public Integer call() throws Exception {
            for (int i = 0; i < num; i++) {
                System.out.println(Thread.currentThread().getName());
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            return num;
        }
    }
    
}
```

输出结果（不固定）：

```
Thread-9
Thread-8
Thread-9
Thread-8
Thread-9
Thread-8
Thread-9
Thread-8
Thread-9
Thread-8
Thread-9
Thread-8
Thread-9
Thread-8
Thread-9
Thread-8
Thread-9
Thread-8
Thread-9
Thread-8
Thread-9
10
Thread-9
Thread-9
Thread-9
Thread-9
Thread-9
Thread-9
Thread-9
Thread-9
Thread-9
20
```

## 三种方式的比较

1. Java中没有多重继承，因此在使用继承Thread类的方式创建线程时，不能再继承其他类；使用Runnable、Callable接口创建多线程时，还可以继承其他类，在这种方式下，多个线程可以共享同一个target对象，所以非常适合多个相同线程来处理同一份资源的情况； 
2. 使用Callable接口可以从线程中获取返回值。

# 线程同步

## synchronized关键字

从1.0版本开始，Java中的每一个对象都有一个内部锁，如果一个方法使用synchronized关键字声明，线程将获得对象的内部锁。
synchronized有两种方式：锁方法、锁对象。

1. 锁方法，即用synchronized关键字修饰方法：

```
// 获得的是对象的内部锁
public synchronized void method(){
	method body
}
```

```
// 获得的是类锁，由于一个class不论被实例化多少次，其中的静态方法和静态变量在内存中都只由一份。所以，一旦一个静态的方法被申明为synchronized。此类所有的实例化对象在调用此方法，共用同一把锁。
public static synchronized void(){
	method body
}
```

2. 锁对象：

```
public void method(){
	// 获得的是对象的内部锁
	synchronized(this){
		code block
	}	
}
```

```
Object obj = new Object();
public void method(){
	// 获得的是obj的内部锁
	synchronized(obj){
		code block
	}
}
```

```
public void method(){
	// 获得的是类锁
	synchronized(xxx.class){
		code block
	}	
}
```

```
public void method(){
	// 获得的是类锁
	synchronized(Class.forName("xxx")){
		code block
	}	
}
```

## Lock

Lock(锁对象)允许把锁的实现作为Java类，而不是作为语言的特性来实现，这就为Lock的多种实现留下了空间，各种实现可能有不同的调度算法、性能特性或者锁定语义。

### 方法摘要

返回值|方法|解释
--|--|--
void|lock()|获取锁。
void|lockInterruptibly()|如果当前线程未被中断，则获取锁。
Condition|newCondition()|返回绑定到此 Lock 实例的新 Condition 实例。
boolean|tryLock()|仅在调用时锁为空闲状态才获取该锁。
boolean|tryLock(long time, TimeUnit unit)|如果锁在给定的等待时间内空闲，并且当前线程未被中断，则获取锁。
void|unlock()|释放锁。

### ReentrantLock

ReentrantLock类是Lock的一个实现，使用ReentrantLock的代码块结构如下：

```
myLock.lock();
try{
	......
} finally {
	myLock.unlock();
}
```

`ReentrantLock(true)可以构造一个带有公平策略的锁，听起来公平锁更合理一些，但是使用公平锁比使用常规锁要慢很多。`

## ReadWriteLock

ReadWriteLock 维护了一对相关的锁，一个用于只读操作，另一个用于写入操作。只要没有 writer，读取锁可以由多个 reader 线程同时保持。写入锁是独占的。
与互斥锁相比，读-写锁允许对共享数据进行更高级别的并发访问。虽然一次只有一个线程（writer 线程）可以修改共享数据，但在许多情况下，任何数量的线程可以同时读取共享数据（reader 线程），读-写锁利用了这一点。从理论上讲，与互斥锁相比，使用读-写锁所允许的并发性增强将带来更大的性能提高。在实践中，只有在多处理器上并且只在访问模式适用于共享数据时，才能完全实现并发性增强。

### 方法摘要
返回值|方法|解释
--|--|--
Lock|readLock()|返回用于读取操作的锁。
Lock|writeLock()|返回用于写入操作的锁。

### ReentrantReadWriteLock

ReentrantReadWriteLock是ReadWriteLock的实现类，其使用方式如下：

```
ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
lock.readLock().lock();		// 获得读锁
lock.readLock().unLock();	// 释放读锁
lock.writeLock().lock();	// 获得写锁
lock.writeLock().unLock();	// 释放写锁
```

下面来看一个实际的例子，使用读写锁来实现查询、存钱、取钱：
例：

```
public class ReadWriteLockStudy {

    static class Account {

        private long balance;

        private ReadWriteLock lock = new ReentrantReadWriteLock();
        private Lock readLock = lock.readLock();
        private Lock writeLock = lock.writeLock();

        public void show() {
            readLock.lock();
            System.out.println("获得readLock -- " + Thread.currentThread().getName());
            try {
                System.out.println("显示余额:" + balance);
            } finally {
                readLock.unlock();
            }
            System.out.println("释放readLock -- " + Thread.currentThread().getName());
        }

        public void put(long money) {
            writeLock.lock();
            System.out.println("获得writeLock -- " + Thread.currentThread().getName());
            try {
                balance += money;
                System.out.println("存入:" + money + ",余额:" + balance);
            } finally {
                writeLock.unlock();
                System.out.println("释放writeLock -- " + Thread.currentThread().getName());
            }
        }

        public void take(long money) {
            writeLock.lock();
            System.out.println("获得writeLock -- " + Thread.currentThread().getName());
            try {
                if (balance < money)
                    System.out.println("余额不足, 需取: " + money + "; 余额: " + balance);
                else {
                    balance -= money;
                    System.out.println("取出:" + money + ",余额:" + balance);
                }
            } finally {
                writeLock.unlock();
                System.out.println("释放writeLock -- " + Thread.currentThread().getName());
            }
        }
    }

    static class PutThread extends Thread {
        private Account account;

        public PutThread(Account account) {
            this.account = account;
        }

        @Override
        public void run() {
            Random random = new Random();
            int i = 0;
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

    static class TakeThread extends Thread {
        private Account account;

        public TakeThread(Account account) {
            this.account = account;
        }

        @Override
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

    static class ShowThread extends Thread {
        private Account account;

        public ShowThread(Account account) {
            this.account = account;
        }

        @Override
        public void run() {
            while (true) {
                account.show();
                try {
                    sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String args[]) {
        Account account = new Account();
        PutThread putThread = new PutThread(account);
        putThread.setName("putThread");
        TakeThread takeThread = new TakeThread(account);
        takeThread.setName("takeThread");
        ShowThread showThread1 = new ShowThread(account);
        showThread1.setName("showThread1");
        ShowThread showThread2 = new ShowThread(account);
        showThread2.setName("showThread2");

        putThread.start();
        takeThread.start();
        showThread1.start();
        showThread2.start();
    }

}
```

输出结果（不固定）：

```
获得writeLock -- putThread
存入:46,余额:46
释放writeLock -- putThread
获得writeLock -- takeThread
余额不足, 需取: 95; 余额: 46
释放writeLock -- takeThread
获得readLock -- showThread1
显示余额:46
释放readLock -- showThread1
获得readLock -- showThread2
显示余额:46
释放readLock -- showThread2
...
...
...
获得readLock -- showThread2
获得readLock -- showThread1
显示余额:46
显示余额:46
释放readLock -- showThread1
释放readLock -- showThread2
```

可以看到读-读不互斥，读-写互斥，写-写互斥。

### Condition

条件（也称为条件队列 或条件变量）为线程提供了一个含义，以便在某个状态条件现在可能为 true 的另一个线程通知它之前，一直挂起该线程（即让其“等待”）。因为访问此共享状态信息发生在不同的线程中，所以它必须受保护，因此要将某种形式的锁与该条件相关联。等待提供一个条件的主要属性是：以原子方式 释放相关的锁，并挂起当前线程，就像 Object.wait 做的那样。
Condition可以替代传统的线程间通信，用await()替换wait()，用signal()替换notify()，用signalAll()替换notifyAll()。
`不直接使用wait(), notify(), notifyAll()是因为 这几个方法是final方法。`
可以为多个线程间建立不同的Condition。

将上面的例子修改一下，增加两个功能：
1. 当余额少于100时，马上存入100；
2. 当余额大于200时，马上取出100；

代码如下：

```
public class ConditionStudy {

    static class Account {

        private long balance;

        private Lock lock = new ReentrantLock();
        private Condition condition100 = lock.newCondition();
        private Condition condition200 = lock.newCondition();

        public void put(long money) {
            lock.lock();
            System.out.println("获得lock -- " + Thread.currentThread().getName());
            try {
                balance += money;
                System.out.println("存入:" + money + ",余额:" + balance + " -- " + Thread.currentThread().getName());
                condition200.signal();
            } finally {
                lock.unlock();
                System.out.println("释放lock -- " + Thread.currentThread().getName());
            }
        }

        public void take(long money) {
            lock.lock();
            System.out.println("获得lock -- " + Thread.currentThread().getName());
            try {
                if (balance < money)
                    System.out.println("余额不足, 需取: " + money + "; 余额: " + balance + " -- " + Thread.currentThread().getName());
                else {
                    balance -= money;
                    System.out.println("取出:" + money + ",余额:" + balance + " -- " + Thread.currentThread().getName());
                    condition100.signal();
                }
            } finally {
                lock.unlock();
                System.out.println("释放lock -- " + Thread.currentThread().getName());
            }
        }

        public void put100() {
            lock.lock();
            System.out.println("获得lock -- " + Thread.currentThread().getName());
            try {
                while (balance >= 100) {
                    System.out.println("余额大于等于100, 等待, 释放锁" + " -- " + Thread.currentThread().getName());
                    condition100.await();
                }
                balance += 100;
                System.out.println("存入100, 余额:" + balance + " -- " + Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
                System.out.println("释放lock -- " + Thread.currentThread().getName());
            }
        }

        public void take100() {
            lock.lock();
            System.out.println("获得lock -- " + Thread.currentThread().getName());
            try {
                while (balance < 200) {
                    System.out.println("余额小于200, 等待, 释放锁" + " -- " + Thread.currentThread().getName());
                    condition200.await();
                }
                balance -= 100;
                System.out.println("取出100, 余额:" + balance + " -- " + Thread.currentThread().getName());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock.unlock();
                System.out.println("释放lock -- " + Thread.currentThread().getName());
            }
        }
    }

    static class PutThread extends Thread {
        private Account account;

        public PutThread(Account account) {
            this.account = account;
        }

        @Override
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

    static class TakeThread extends Thread {
        private Account account;

        public TakeThread(Account account) {
            this.account = account;
        }

        @Override
        public void run() {
            Random random = new Random();
            int i = 0;
            while (i++ < 10) {
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

    static class Put100Thread extends Thread {
        private Account account;

        public Put100Thread(Account account) {
            this.account = account;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    account.put100();
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    static class Take100Thread extends Thread {
        private Account account;

        public Take100Thread(Account account) {
            this.account = account;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    account.take100();
                    sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String args[]) {
        Account account = new Account();
        PutThread putThread = new PutThread(account);
        putThread.setName("putThread");
        TakeThread takeThread = new TakeThread(account);
        takeThread.setName("takeThread");
        Put100Thread put100Thread = new Put100Thread(account);
        put100Thread.setName("put100Thread");
        Take100Thread take100Thread = new Take100Thread(account);
        take100Thread.setName("take100Thread");

        putThread.start();
        takeThread.start();
        put100Thread.start();
        take100Thread.start();
    }

}
```

## volatile

volatile可以保证每次都从主内存中读取数据，且每次数据修改都写回主内存。如果一个变量声明为volatile，那么编译器和虚拟机就知道该域是可能被另一个线程并发更新的。

## 死锁

死锁是指多个线程相互等待它方占有的资源而导致的每个线程都无法执行下去的情况。
下面是一个简单的死锁例子：

```
public class DeadLock {

    static class MemStore {

        Lock lock1 = new ReentrantLock();
        Lock lock2 = new ReentrantLock();

        public void method1() {
            lock1.lock();
            System.out.println("获得lock1 -- " + Thread.currentThread().getName());
            try {
                Thread.sleep(1000);
                lock2.lock();
                try {
                    System.out.println("获得lock2 -- " + Thread.currentThread().getName());
                } finally {
                    lock2.unlock();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock1.unlock();
            }
        }

        public void method2() {
            lock2.lock();
            System.out.println("获得lock2 -- " + Thread.currentThread().getName());
            try {
                Thread.sleep(1000);
                lock1.lock();
                try {
                    System.out.println("获得lock1 -- " + Thread.currentThread().getName());
                } finally {
                    lock1.unlock();
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                lock2.unlock();
            }
        }
    }

    static class Thread1 extends Thread {
        private MemStore memStore;

        public Thread1(MemStore memStore) {
            this.memStore = memStore;
        }

        @Override
        public void run(){
            memStore.method1();
        }
    }

    static class Thread2 extends Thread {
        private MemStore memStore;

        public Thread2(MemStore memStore) {
            this.memStore = memStore;
        }

        @Override
        public void run(){
            memStore.method2();
        }
    }

    public static void main(String args[]) {
        MemStore memStore = new MemStore();
        Thread1 thread1 = new Thread1(memStore);
        thread1.setName("thread1");
        Thread2 thread2 = new Thread2(memStore);
        thread2.setName("thread2");
        thread1.start();
        thread2.start();
    }

}
```

输出结果：

```
获得lock1 -- thread1
获得lock2 -- thread2
```
可以看到thread1获得了lock1，等待获得lock2，而thread2获得了lock2，等待获得lock1，从而死锁。

# 线程池

多线程技术主要解决处理器单元内多个线程执行的问题，它可以显著减少处理器单元的闲置时间，增加处理器单元的吞吐能力。

## 线程池原理

假设一个服务器完成一项任务所需时间为：T1 创建线程时间，T2 在线程中执行任务的时间，T3 销毁线程时间。如果：T1 + T3 远大于 T2，则可以采用线程池，以提高服务器性能。

一个线程池包括以下四个基本组成部分：

1. 线程池管理器（ThreadPool）：用于创建并管理线程池，包括 创建线程池，销毁线程池，添加新任务；
2. 工作线程（PoolWorker）：线程池中线程，在没有任务时处于等待状态，可以循环的执行任务；
3. 任务接口（Task）：每个任务必须实现的接口，以供工作线程调度任务的执行，它主要规定了任务的入口，任务执行完后的收尾工作，任务的执行状态等；
4. 任务队列（taskQueue）：用于存放没有处理的任务。提供一种缓冲机制。

线程池技术正是关注如何缩短或调整T1,T3时间的技术，从而提高服务器程序性能的。它把T1，T3分别安排在服务器程序的启动和结束的时间段或者一些空闲的时间段，这样在服务器程序处理客户请求时，不会有T1，T3的开销了。

## 简单线程池的实现

```
public class SimpleThreadPool {

    // 线程数
    private int threadNum = 10;
    // 工作线程
    private WorkThread[] workThreads;
    // 任务队列, 待执行的线程
    private BlockingQueue<Runnable> taskQueue;

    public SimpleThreadPool(int threadNum) {
        if (threadNum > 0)
            this.threadNum = threadNum;
        // 初始化任务队列
        taskQueue = new LinkedBlockingDeque<>();
        // 初始化工作线程
        workThreads = new WorkThread[this.threadNum];
        int i = 0;
        while (i < threadNum) {
            workThreads[i] = new WorkThread();
            workThreads[i].setName("workThread-" + i);
            // 启动工作线程
            workThreads[i].start();
            i++;
        }
    }

    public void execute(Runnable runnable) {
        taskQueue.add(runnable);
    }

    public void execute(Runnable[] runnableList) {
        for (Runnable runnable : runnableList)
            execute(runnable);
    }

    public void destroy(){
        for(WorkThread workThread: workThreads)
            workThread.stopRun();
    }

    class WorkThread extends Thread {
        private volatile boolean runFlag = true;

        @Override
        public void run() {
            while (runFlag)
                try {
                    Runnable task = taskQueue.take();
                    task.run();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
        }

        public void stopRun() {
            runFlag = false;
        }
    }

    private static class Task implements Runnable {

        private String name;

        public Task(String name) {
            this.name = name;
        }

        @Override
        public void run() {
            System.out.println(name + " run." + "current thread: " + Thread.currentThread().getName());
        }
    }

    public static void main(String args[]) throws InterruptedException {
        SimpleThreadPool simpleThreadPool = new SimpleThreadPool(3);
        simpleThreadPool.execute(new Task("task0"));
        simpleThreadPool.execute(new Runnable[]{new Task("task1"), new Task("task2"), new Task("task3"), new Task("task4")});
        simpleThreadPool.execute(new Runnable[]{new Task("task4"), new Task("task5"), new Task("task6")});
        Thread.sleep(1000);
        simpleThreadPool.destroy();
    }

}
```

输出结果（不固定）：

```
task0 run.current thread: workThread-0
task2 run.current thread: workThread-2
task1 run.current thread: workThread-1
task3 run.current thread: workThread-0
task4 run.current thread: workThread-2
task4 run.current thread: workThread-1
task5 run.current thread: workThread-0
task6 run.current thread: workThread-2
```

# concurrent包

java.util.concurrent包是在并发编程中很常用的实用工具包。此包包括了几个小的、已标准化的可扩展框架，以及一些提供有用功能的类，没有这些类，这些功能会很难实现或实现起来冗长乏味。下面简要描述主要的组件。

## 执行器

接口。Executor是一个简单的标准化接口，用于定义类似于线程的自定义子系统，包括线程池、异步IO和轻量级任务框架。根据所使用的具体Executor类的不同，可能在新创建的线程中，现有的任务执行线程中，或者调用execute()的线程中执行任务，并且可能顺序或并发执行。ExecutorService 提供了多个完整的异步任务执行框架。ExecutorService 管理任务的排队和安排，并允许受控制的关闭。ScheduledExecutorService 子接口及相关的接口添加了对延迟的和定期任务执行的支持。ExecutorService 提供了安排异步执行的方法，可执行由 Callable 表示的任何函数，结果类似于 Runnable。Future 返回函数的结果，允许确定执行是否完成，并提供取消执行的方法。RunnableFuture 是拥有 run 方法的 Future，run 方法执行时将设置其结果。

实现。类 ThreadPoolExecutor 和 ScheduledThreadPoolExecutor 提供可调的、灵活的线程池。Executors 类提供大多数 Executor 的常见类型和配置的工厂方法，以及使用它们的几种实用工具方法。其他基于 Executor 的实用工具包括具体类 FutureTask，它提供 Future 的常见可扩展实现，以及 ExecutorCompletionService，它有助于协调对异步任务组的处理。

## 队列

java.util.concurrent.ConcurrentLinkedQueue类提供了高效的、可伸缩的、线程安全的非阻塞 FIFO 队列。java.util.concurrent 中的五个实现都支持扩展的 BlockingQueue 接口，该接口定义了 put 和 take 的阻塞版本：LinkedBlockingQueue、ArrayBlockingQueue、SynchronousQueue、PriorityBlockingQueue 和 DelayQueue。这些不同的类覆盖了生产者-使用者、消息传递、并行任务执行和相关并发设计的大多数常见使用的上下文。BlockingDeque 接口扩展 BlockingQueue，以支持 FIFO 和 LIFO（基于堆栈）操作。LinkedBlockingDeque 类提供一个实现。

## 计时

TimeUnit 类为指定和控制基于超时的操作提供了多重粒度（包括纳秒级）。该包中的大多数类除了包含不确定的等待之外，还包含基于超时的操作。在使用超时的所有情况中，超时指定了在表明已超时前该方法应该等待的最少时间。在超时发生后，实现会“尽力”检测超时。但是，在检测超时与超时之后再次实际执行线程之间可能要经过不确定的时间。接受超时期参数的所有方法将小于等于 0 的值视为根本不会等待。要“永远”等待，可以使用 Long.MAX_VALUE 值。

## 同步器

四个类可协助实现常见的专用同步语句。Semaphore 是一个经典的并发工具。CountDownLatch 是一个极其简单但又极其常用的实用工具，用于在保持给定数目的信号、事件或条件前阻塞执行。CyclicBarrier 是一个可重置的多路同步点，在某些并行编程风格中很有用。Exchanger 允许两个线程在 collection 点交换对象，它在多流水线设计中是有用的。

## 并发 Collection

除队列外，此包还提供了设计用于多线程上下文中的 Collection 实现：ConcurrentHashMap、ConcurrentSkipListMap、ConcurrentSkipListSet、CopyOnWriteArrayList 和 CopyOnWriteArraySet。当期望许多线程访问一个给定 collection 时，ConcurrentHashMap 通常优于同步的 HashMap，ConcurrentSkipListMap 通常优于同步的 TreeMap。当期望的读数和遍历远远大于列表的更新数时，CopyOnWriteArrayList 优于同步的 ArrayList。
此包中与某些类一起使用的“Concurrent&rdquo前缀;是一种简写，表明与类似的“同步”类有所不同。例如，java.util.Hashtable 和 Collections.synchronizedMap(new HashMap()) 是同步的，但 ConcurrentHashMap 则是“并发的”。并发 collection 是线程安全的，但是不受单个排他锁的管理。在 ConcurrentHashMap 这一特定情况下，它可以安全地允许进行任意数目的并发读取，以及数目可调的并发写入。需要通过单个锁不允许对 collection 的所有访问时，“同步”类是很有用的，其代价是较差的可伸缩性。在期望多个线程访问公共 collection 的其他情况中，通常“并发”版本要更好一些。当 collection 是未共享的，或者仅保持其他锁时 collection 是可访问的情况下，非同步 collection 则要更好一些。

大多数并发 Collection 实现（包括大多数 Queue）与常规的 java.util 约定也不同，因为它们的迭代器提供了弱一致的，而不是快速失败的遍历。弱一致的迭代器是线程安全的，但是在迭代时没有必要冻结 collection，所以它不一定反映自迭代器创建以来的所有更新。

## 内存一致性属性

只有写入操作 happen-before 读取操作时，才保证一个线程写入的结果对另一个线程的读取是可视的。synchronized 和 volatile 构造 happen-before 关系，Thread.start() 和 Thread.join() 方法形成 happen-before 关系。尤其是：
线程中的每个操作 happen-before 稍后按程序顺序传入的该线程中的每个操作。
一个解除锁监视器的（synchronized 阻塞或方法退出）happen-before 相同监视器的每个后续锁（synchronized 阻塞或方法进入）。并且因为 happen-before 关系是可传递的，所以解除锁定之前的线程的所有操作 happen-before 锁定该监视器的任何线程后续的所有操作。
写入 volatile 字段 happen-before 每个后续读取相同字段。volatile 字段的读取和写入与进入和退出监视器具有相似的内存一致性效果，但不 需要互斥锁。
在线程上调用 start happen-before 已启动的线程中的任何线程。
线程中的所有操作 happen-before 从该线程上的 join 成功返回的任何其他线程。
java.util.concurrent 中所有类的方法及其子包扩展了这些对更高级别同步的保证。尤其是：
线程中将一个对象放入任何并发 collection 之前的操作 happen-before 从另一线程中的 collection 访问或移除该元素的后续操作。
线程中向 Executor 提交 Runnable 之前的操作 happen-before 其执行开始。同样适用于向 ExecutorService 提交 Callables。
异步计算（由 Future 表示）所采取的操作 happen-before 通过另一线程中 Future.get() 获取结果后续的操作。
“释放”同步储存方法（如 Lock.unlock、Semaphore.release 和 CountDownLatch.countDown）之前的操作 happen-before 另一线程中相同同步储存对象成功“获取”方法（如 Lock.lock、Semaphore.acquire、Condition.await 和 CountDownLatch.await）的后续操作。
对于通过 Exchanger 成功交换对象的每个线程对，每个线程中 exchange() 之前的操作 happen-before 另一线程中对应 exchange() 后续的操作。
调用 CyclicBarrier.await 之前的操作 happen-before 屏障操作所执行的操作，屏障操作所执行的操作 happen-before 从另一线程中对应 await 成功返回的后续操作。

# 关于线程个数

我们的应用程序应该创建多少个线程会使得性能得到比较好的提升呢？以下代码可以计算一个合适的线程个数：

```
public abstract class PoolSizeCalculator {

    /**
     * The sample queue size to calculate the size of a single {@link Runnable} element.
     */
    private final int SAMPLE_QUEUE_SIZE = 1000;

    /**
     * Accuracy of test run. It must finish within 20ms of the testTime otherwise we retry the test. This could be
     * configurable.
     */
    private final int EPSYLON = 20;

    /**
     * Control variable for the CPU time investigation.
     */
    private volatile boolean expired;

    /**
     * Time (millis) of the test run in the CPU time calculation.
     */
    private final long testtime = 3000;

    /**
     * Calculates the boundaries of a thread pool for a given {@link Runnable}.
     *
     * @param targetUtilization    the desired utilization of the CPUs (0 <= targetUtilization <= 1)
     * @param targetQueueSizeBytes the desired maximum work queue size of the thread pool (bytes)
     */
    protected void calculateBoundaries(BigDecimal targetUtilization, BigDecimal targetQueueSizeBytes) {
        calculateOptimalCapacity(targetQueueSizeBytes);
        Runnable task = createTask();
        start(task);
        start(task); // warm up phase
        long cputime = getCurrentThreadCPUTime();
        start(task); // test intervall
        cputime = getCurrentThreadCPUTime() - cputime;
        long waittime = (testtime * 1000000) - cputime;
        calculateOptimalThreadCount(cputime, waittime, targetUtilization);
    }

    private void calculateOptimalCapacity(BigDecimal targetQueueSizeBytes) {
        long mem = calculateMemoryUsage();
        BigDecimal queueCapacity = targetQueueSizeBytes.divide(new BigDecimal(mem), RoundingMode.HALF_UP);
        System.out.println("Target queue memory usage (bytes): " + targetQueueSizeBytes);
        System.out.println("createTask() produced " + createTask().getClass().getName() + " which took " + mem
                + " bytes in a queue");
        System.out.println("Formula: " + targetQueueSizeBytes + " / " + mem);
        System.out.println("* Recommended queue capacity (bytes): " + queueCapacity);
    }

    /**
     * Brian Goetz' optimal thread count formula, see 'Java Concurrency in Practice' (chapter 8.2)
     *
     * @param cpu               cpu time consumed by considered task
     * @param wait              wait time of considered task
     * @param targetUtilization target utilization of the system
     */
    private void calculateOptimalThreadCount(long cpu, long wait, BigDecimal targetUtilization) {
        BigDecimal waitTime = new BigDecimal(wait);
        BigDecimal computeTime = new BigDecimal(cpu);
        BigDecimal numberOfCPU = new BigDecimal(Runtime.getRuntime().availableProcessors());
        BigDecimal optimalthreadcount = numberOfCPU.multiply(targetUtilization).multiply(
                new BigDecimal(1).add(waitTime.divide(computeTime, RoundingMode.HALF_UP)));
        System.out.println("Number of CPU: " + numberOfCPU);
        System.out.println("Target utilization: " + targetUtilization);
        System.out.println("Elapsed time (nanos): " + (testtime * 1000000));
        System.out.println("Compute time (nanos): " + cpu);
        System.out.println("Wait time (nanos): " + wait);
        System.out.println("Formula: " + numberOfCPU + " * " + targetUtilization + " * (1 + " + waitTime + " / "
                + computeTime + ")");
        System.out.println("* Optimal thread count: " + optimalthreadcount);
    }

    /**
     * Runs the {@link Runnable} over a period defined in {@link #testtime}. Based on Heinz Kabbutz' ideas
     * (http://www.javaspecialists.eu/archive/Issue124.html).
     *
     * @param task the runnable under investigation
     */
    public void start(Runnable task) {
        long start = 0;
        int runs = 0;
        do {
            if (++runs > 5) {
                throw new IllegalStateException("Test not accurate");
            }
            expired = false;
            start = System.currentTimeMillis();
            Timer timer = new Timer();
            timer.schedule(new TimerTask() {
                public void run() {
                    expired = true;
                }
            }, testtime);
            while (!expired) {
                task.run();
            }
            start = System.currentTimeMillis() - start;
            timer.cancel();
        } while (Math.abs(start - testtime) > EPSYLON);
        collectGarbage(3);
    }

    private void collectGarbage(int times) {
        for (int i = 0; i < times; i++) {
            System.gc();
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    /**
     * Calculates the memory usage of a single element in a work queue. Based on Heinz Kabbutz' ideas
     * (http://www.javaspecialists.eu/archive/Issue029.html).
     *
     * @return memory usage of a single {@link Runnable} element in the thread pools work queue
     */
    public long calculateMemoryUsage() {
        BlockingQueue<Runnable> queue = createWorkQueue();
        for (int i = 0; i < SAMPLE_QUEUE_SIZE; i++) {
            queue.add(createTask());
        }
        long mem0 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
        long mem1 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
        queue = null;
        collectGarbage(15);
        mem0 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
        queue = createWorkQueue();
        for (int i = 0; i < SAMPLE_QUEUE_SIZE; i++) {
            queue.add(createTask());
        }
        collectGarbage(15);
        mem1 = Runtime.getRuntime().totalMemory() - Runtime.getRuntime().freeMemory();
        return (mem1 - mem0) / SAMPLE_QUEUE_SIZE;
    }

    /**
     * Create your runnable task here.
     *
     * @return an instance of your runnable task under investigation
     */
    protected abstract Runnable createTask();

    /**
     * Return an instance of the queue used in the thread pool.
     *
     * @return queue instance
     */
    protected abstract BlockingQueue<Runnable> createWorkQueue();

    /**
     * Calculate current cpu time. Various frameworks may be used here, depending on the operating system in use. (e.g.
     * http://www.hyperic.com/products/sigar). The more accurate the CPU time measurement, the more accurate the results
     * for thread count boundaries.
     *
     * @return current cpu time of current thread
     */
    protected abstract long getCurrentThreadCPUTime();

}
```

```
public class MyPoolSizeCalculator extends PoolSizeCalculator {

    public static void main(String[] args) throws InterruptedException,
            InstantiationException,
            IllegalAccessException,
            ClassNotFoundException {
        MyPoolSizeCalculator calculator = new MyPoolSizeCalculator();
        calculator.calculateBoundaries(new BigDecimal(1.0),
                new BigDecimal(100000));
    }

    protected long getCurrentThreadCPUTime() {
        return ManagementFactory.getThreadMXBean().getCurrentThreadCpuTime();
    }

    protected Runnable createTask() {
        return new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10000; i++)
                    System.out.println(i);
            }
        };
    }

    protected BlockingQueue<Runnable> createWorkQueue() {
        return new LinkedBlockingQueue<>();
    }

}
```

`createTask的方法根据自己的实际需要进行修改。`

得到类似如下结果 ：

```
Number of CPU: 4
Target utilization: 1
Elapsed time (nanos): 3000000000
Compute time (nanos): 2017495000
Wait time (nanos): 982505000
Formula: 4 * 1 * (1 + 982505000 / 2017495000)
* Optimal thread count: 4
```
由此可知，在上述场景中，创建4个线程比较合适。
