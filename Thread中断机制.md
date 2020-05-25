# Thread中断机制

---
[参考链接：Java Thread的interrupt详解](https://blog.csdn.net/paincupid/article/details/47626819)

[参考链接：Thread的中断机制(interrupt)](https://www.cnblogs.com/onlywujun/p/3565082.html)

---

+ ***`没有任何语言方面的需求一个被中断的线程应该终止`***
+ ***`中断一个线程只是为了引起该线程的注意，被中断线程可以决定如何应对中断`***
+ ***`Thread.interrupt()方法不会中断一个正在运行的线程`***
+ 如果线程在调用 Object 类的 wait()、wait(long) 或 wait(long, int) 方法，或者该类的 join()、join(long)、join(long, int)、sleep(long) 或 sleep(long, int) 方法过程中受阻，则其中断状态将被清除，它还将收到一个InterruptedException异常，这个时候，可以通过捕获InterruptedException异常来终止线程的执行，具体可以通过return等退出或改变共享变量的值使其退出（`即当线程被置为中断状态时，试图调用上述方法，即会抛出异常，并清除中断状态`）
+ `synchronized在获锁的过程中是不能被中断的，意思是说如果产生了死锁，则不可能被中断`
+ 如果该线程在可中断的通道上的 I/O 操作中受阻，则该通道将被关闭，该线程的中断状态将被清除并且该线程将收到一个 ClosedByInterruptException

## 有关线程中断的3个方法

```java
// 中断线程（实例方法）
public void Thread.interrupt();

// 判断线程是否被中断（实例方法）
public boolean Thread.isInterrupted();

// 判断是否被中断并清除当前中断状态（静态方法）
public static boolean Thread.interrupted();
```

## 中断线程

线程的thread.interrupt()方法是中断线程，将会设置该线程的中断状态位，即设置为true，中断的结果线程是死亡、还是等待新的任务或是继续运行至下一步，就`取决于这个程序本身`

***`更普遍的情况是，一个线程将把中断看作一个终止请求`***

## 判断线程是否被中断

判断某个线程是否已被发送过中断请求，使用 `Thread.currentThread().isInterrupted()` 方法（因为它将线程中断标示位设置为true后，不会立刻清除中断标示位，即不会将中断标设置为false
`thread.interrupted()` 该方法调用后会将中断标示位清除，即重新设置为false

## 如何中断线程

如果一个线程处于了阻塞状态（如线程调用了`thread.sleep、thread.join、thread.wait、1.5中的condition.await、以及可中断的通道上的 I/O 操作方法`后可进入阻塞状态），则在线程在检查中断标示时如果发现中断标示为true，则会在这些阻塞方法（sleep、join、wait、1.5中的condition.await及可中断的通道上的 I/O 操作方法）调用处抛出InterruptedException异常，并且在抛出异常后立即将线程的中断标示位清除，即重新设置为false
`抛出异常是为了线程从阻塞状态醒过来，并在结束线程前让程序员有足够的时间来处理中断请求`

`synchronized在获锁的过程中是不能被中断的`，意思是说如果产生了死锁，则不可能被中断，可参考以下例子:

```java
package top.tjsanshao;

import java.util.concurrent.TimeUnit;

/**
 * thread test
 *
 * @author TjSanshao
 * @date 2020-05-13 15:30
 */
public class ThreadTest implements Runnable {
    private Object l1;
    private Object l2;

    public ThreadTest(Object l1, Object l2) {
        this.l1 = l1;
        this.l2 = l2;
    }

    static void deathLock(Object lock1, Object lock2) {
        try {
            synchronized (lock1) {
                TimeUnit.SECONDS.sleep(1);
                synchronized (lock2) {
                    // 需要2个锁
                    System.out.println(Thread.currentThread());
                }
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        deathLock(this.l1, this.l2);
    }

    public static void main(String[] args) throws InterruptedException {
        String lock1 = "lock1";
        String lock2 = "lock2";
        Thread t1 = new Thread(new ThreadTest(lock1, lock2));
        t1.start();
        Thread t2 = new Thread(new ThreadTest(lock2, lock1));
        t2.start();
        TimeUnit.SECONDS.sleep(3);
        // 3秒后，2个线程都各自获取了1个锁，都在等待另一个锁
        t1.interrupt();
        t2.interrupt();
        System.out.println("" + t1.getId() + t1.isInterrupted());
        System.out.println("" + t2.getId() + t2.isInterrupted());
        TimeUnit.SECONDS.sleep(10);
    }
}

```

以上代码中的2个线程会一直等待锁，无法中断

## 中断应用

### 使用中断信号量中断非阻塞状态的线程

中断线程最好的，最受推荐的方式是，使用共享变量（shared variable）发出信号，告诉线程必须停止正在运行的任务
线程必须周期性的核查这一变量，然后有秩序地中止任务

```java
class Example2 extends Thread {
    volatile boolean stop = false;// 线程中断信号量

    public static void main(String args[]) throws Exception {
        Example2 thread = new Example2();
        System.out.println("Starting thread...");
        thread.start();
        Thread.sleep(3000);
        System.out.println("Asking thread to stop...");
        // 设置中断信号量
        thread.stop = true;
        Thread.sleep(3000);
        System.out.println("Stopping application...");
    }

    public void run() {
        // 每隔一秒检测一下中断信号量
        while (!stop) {
            System.out.println("Thread is running...");
            long time = System.currentTimeMillis();
            /*
             * 使用while循环模拟 sleep 方法，这里不要使用sleep，否则在阻塞时会 抛
             * InterruptedException异常而退出循环，这样while检测stop条件就不会执行，
             * 失去了意义。
             */
            while ((System.currentTimeMillis() - time < 1000)) {}
        }
        System.out.println("Thread exiting under request...");
    }
}
```

### 使用thread.interrupt()中断非阻塞状态线程

```java
class Example2 extends Thread {
    public static void main(String args[]) throws Exception {
        Example2 thread = new Example2();
        System.out.println("Starting thread...");
        thread.start();
        Thread.sleep(3000);
        System.out.println("Asking thread to stop...");
        // 发出中断请求
        thread.interrupt();
        Thread.sleep(3000);
        System.out.println("Stopping application...");
    }

    public void run() {
        // 每隔一秒检测是否设置了中断标示
        while (!Thread.currentThread().isInterrupted()) {
            System.out.println("Thread is running...");
            long time = System.currentTimeMillis();
            // 使用while循环模拟 sleep
            while ((System.currentTimeMillis() - time < 1000) ) {
            }
        }
        System.out.println("Thread exiting under request...");
    }
}
```

`注意，如果线程被阻塞，它便不能核查共享变量，也就不能停止`

### 使用thread.interrupt()中断阻塞状态线程

`Thread.interrupt()方法不会中断一个正在运行的线程`
这一方法实际上完成的是，设置线程的中断标示位，在线程受到阻塞的地方（如调用sleep、wait、join等地方）抛出一个异常InterruptedException，并且中断状态也将被清除，这样线程就得以退出阻塞的状态

```java
class Example3 extends Thread {
    public static void main(String args[]) throws Exception {
        Example3 thread = new Example3();
        System.out.println("Starting thread...");
        thread.start();
        Thread.sleep(3000);
        System.out.println("Asking thread to stop...");
        thread.interrupt();// 等中断信号量设置后再调用
        Thread.sleep(3000);
        System.out.println("Stopping application...");
    }

    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            System.out.println("Thread running...");
            try {
                /*
                 * 如果线程阻塞，将不会去检查中断信号量stop变量，所 以thread.interrupt()
                 * 会使阻塞线程从阻塞的地方抛出异常，让阻塞线程从阻塞状态逃离出来，并
                 * 进行异常块进行 相应的处理
                 */
                Thread.sleep(1000);// 线程阻塞，如果线程收到中断操作信号将抛出异常
            } catch (InterruptedException e) {
                System.out.println("Thread interrupted...");
                /*
                 * 如果线程在调用 Object.wait()方法，或者该类的 join() 、sleep()方法
                 * 过程中受阻，则其中断状态将被清除
                 */
                System.out.println(this.isInterrupted());// false

                // 中不中断由自己决定，如果需要真真中断线程，则需要重新设置中断位，如果
                // 不需要，则不用调用
                Thread.currentThread().interrupt();
            }
        }
        System.out.println("Thread exiting under request...");
    }
}
```

一旦Example3中的Thread.interrupt()被调用，线程便收到一个异常，于是逃离了阻塞状态并确定应该停止

`Object.wait, Thread.sleep方法，会不断的轮询监听 interrupted 标志位，发现其设置为true后，会停止阻塞并抛出 InterruptedException异常`

## 有关Java线程中断的概括

+ 没有任何语言方面的需求一个被中断的线程应该终止
+ 中断一个线程只是为了引起该线程的注意，`被中断线程可以决定如何应对中断`
+ 对于处于sleep，join等操作的线程，如果被调用interrupt()后，会抛出InterruptedException，然后`线程的中断标志位会由true重置为false`，`因为线程为了处理异常已经重新处于就绪状态`
+ 不可中断的操作，包括进入synchronized段以及Lock.lock()，inputSteam.read()等，调用interrupt()对于这几个问题无效，`因为它们都不抛出中断异常`，`如果拿不到资源，它们会无限期阻塞下去`
+ 对于Lock.lock()，可以改用`Lock.lockInterruptibly()`，可被中断的加锁操作，它可以抛出中断异常，等同于等待时间无限长的Lock.tryLock(long time, TimeUnit unit)，对于inputStream等资源，有些(实现了`interruptibleChannel`接口)可以通过close()方法将资源关闭，对应的阻塞也会被放开
+ Java的中断是一种协作机制，也就是说调用线程对象的interrupt方法并不一定就中断了正在运行的线程，它只是要求线程自己在合适的时机中断自己，每个线程都有一个boolean的中断状态（这个状态不在Thread的属性上），`interrupt方法仅仅只是将该状态置为true`
+ 一般说来，如果一个方法声明抛出`InterruptedException`，表示该方法是可中断的,比如wait,sleep,join，也就是说可中断方法会对interrupt调用做出响应（例如sleep响应interrupt的操作包括清除中断状态，抛出InterruptedException），异常都是由可中断方法自己抛出来的，并不是直接由interrupt方法直接引起的
