# Reference

---

[参考链接：Java弱引用(WeakReference)的理解与使用](https://www.cnblogs.com/KKKEr/p/10311229.html)

---

## WeakReference

+ 在Java里，当一个对象o被创建时，它被放在Heap里，当GC运行的时候，如果发现没有任何引用指向o，o就会被回收以腾出内存空间，或者换句话说，一个对象被回收，必须满足两个条件: 1)没有任何引用指向它；2)GC被运行
+ 手动置空对象对于程序员来说，是一件繁琐且违背自动回收的理念的
+ 在java中，对于简单对象，当调用它的方法执行完毕后，指向它的引用会被从stack中popup，所以就能在下一次GC执行时被回收了
+ 当一个对象仅仅被weak reference指向，而`没有任何其他strong reference指向的时候，如果GC运行，那么这个对象就会被回收`（可以认为显式声明的引用为强引用，弱引用需要使用WeakReference显式声明）
+ `WeakReference的一个特点是它何时被回收是不可确定的，因为这是由GC运行的不确定性所确定的`
+ `一般用weak reference引用的对象是有价值被cache，而且很容易被重新被构建，且很消耗内存的对象`

## WeakReference ReferenceQueue

+ 在weak reference指向的对象被回收后，weak reference本身其实也就没有用了
+ java提供了一个ReferenceQueue来保存这些所指向的对象已经被回收的reference
+ 在WeakReference构造器中传入该参数

## SoftReference

+ soft reference和weak reference一样，但被GC回收的时候需要多一个条件：系统内存不足时，soft reference指向的object才会被回收
+ soft reference比weak reference更加适合做cache objects的reference，因为它可以尽可能的retain cached objects，减少重建所需的时间和消耗

## StrongReference

+ 垃圾回收器绝不会回收
+ 当内存空间不足，Java 虚拟机宁愿抛出 OutOfMemoryError 错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足问题

## PhantomReference

+ "虚引用"顾名思义，就是形同虚设
+ 与其他几种引用都不同，虚引用并不会决定对象的生命周期
+ 如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收
+ 为了被虚引用关联的对象在被垃圾回收器回收时，能够收到一个系统通知

***虚引用主要用来跟踪对象被垃圾回收的活动***

**虚引用与软引用和弱引用的一个区别在于**：**虚引用必须和引用队列（ReferenceQueue）联合使用**，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中，程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收，程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动

## ReferenceQueue

+ 引用队列可以配合软引用、弱引用及虚引用使用，当引用的对象将要被JVM回收时，会将其加入到引用队列中
+ 一般情况下，通过引用队列可以了解JVM垃圾回收情况

***特别注意，在程序设计中一般很少使用弱引用与虚引用，使用软引用的情况较多***

***软引用可以加速 JVM 对垃圾内存的回收速度，可以维护系统的运行安全，防止内存溢出（OutOfMemory）等问题的产生***
