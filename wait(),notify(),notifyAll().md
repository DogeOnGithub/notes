# wait(),notify(),notifyAll()

---

[参考链接：Java多线程学习之wait、notify/notifyAll 详解](https://www.cnblogs.com/moongeek/p/7631447.html)

---

+ wait()、notify/notifyAll() 方法是Object的本地final方法，无法被重写
+ wait()使当前线程阻塞，前提是`必须先获得锁`，一般配合synchronized 关键字使用，即，一般在synchronized 同步代码块里使用 wait()、notify/notifyAll() 方法
+ 由于 wait()、notify/notifyAll() 在synchronized 代码块执行，说明当前线程一定是获取了锁的
  + 当线程执行wait()方法时候，会释放当前的锁，然后让出CPU，进入等待状态
  + 只有当 notify/notifyAll() 被执行时候，才会唤醒一个或多个正处于等待状态的线程，然后继续往下执行，直到执行完synchronized 代码块的代码或是中途遇到wait() ，再次释放锁（这里是指等待调用notify()/notifyAll()方法的对象锁的线程，可能会存在一个或多个线程同时等待该对象锁）
  + `notify/notifyAll() 的执行只是唤醒沉睡的线程，而不会立即释放锁，锁的释放要看代码块的具体执行情况`（即当前线程仍然持有锁，但是等待该对象锁的其他线程已经处于唤醒状态，等待获得锁）
  + 因此，在实际使用中，尽量在使用了notify/notifyAll() 后立即退出临界区，以唤醒其他线程让其获得锁
+ wait() 需要被try catch包围，以便发生异z常中断也可以使wait等待的线程唤醒
+ notify 和wait 的顺序不能错，如果A线程先执行notify方法，B线程在执行wait方法，那么B线程是无法被唤醒的
+ notify方法只唤醒一个等待（对象的）线程并使该线程开始执行
+ 如果有多个线程等待一个对象，notify只会唤醒其中一个线程，选择哪个线程取决于操作系统对多线程管理的实现
+ notifyAll 会唤醒所有等待(对象的)线程，尽管哪一个线程将会第一个处理取决于操作系统的实现
+ 如果当前情况下有多个线程需要被唤醒，推荐使用notifyAll 方法
