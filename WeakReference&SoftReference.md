# WeakReference & SoftReference

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
+ 在构造器中传入该参数

## SoftReference

+ soft reference和weak reference一样，但被GC回收的时候需要多一个条件：系统内存不足时，soft reference指向的object才会被回收
+ soft reference比weak reference更加适合做cache objects的reference，因为它可以尽可能的retain cached objects，减少重建所需的时间和消耗
