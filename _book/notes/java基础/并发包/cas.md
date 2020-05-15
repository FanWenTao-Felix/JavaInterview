### cas是什么，他会产生什么问题（ABA问题的解决，如加入修改次数、版本号），简述AQS的实现原理

​	CAS(Compare And Swap)，即比较并交换。是解决多线程并行情况下使用锁造成性能损耗的一种机制。
​	CAS是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，并且原语的**执行必须是连续的**，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令，不会造成所谓的数据不一致问题。

> CAS操作包含三个操作数——内存位置（V）、预期原值（A）和新值(B)。
> 如果内存位置V的值与预期原值A相匹配，那么处理器会自动将内存位置V的值更新为新值B；否则，处理器不做任何操作。无论哪种情况，它都会在CAS指令之前返回该位置的值。
> CAS有效地说明了“我认为位置V应该包含值A；如果包含该值，则将B放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。

​	在JAVA中，sun.misc.Unsafe 类提供了硬件级别的原子操作来实现这个CAS。 java.util.concurrent 包下的大量类都使用了这个 Unsafe.java 类的CAS操作。
​	Unsafe类内部方法可以像C的指针一样直接操作内存，从名称看来就可以知道该类是非安全的，毕竟Unsafe拥有着类似于C的指针操作，因此总是不应该首先使用Unsafe类，Java官方也不建议直接使用的Unsafe类。
​	Java中CAS操作的执行依赖于Unsafe类的方法，注意Unsafe类中的所有方法都是native修饰的，也就是说Unsafe类中的方法都直接调用操作系统底层资源执行相应任务。

**CAS 操作相关方法：**
先比较再更新：第一个参数o为给定对象，offset为对象内存的偏移量，通过这个偏移量迅速定位字段并设置或获取该字段的值，expected表示期望值，x表示要设置的值。
下面3个方法都通过CAS原子指令执行操作：

> public final native boolean compareAndSwapObject(Object o, long offset,Object expected, Object x);
> public final native boolean compareAndSwapInt(Object o, long offset,Object expected, Object x);
> public final native boolean compareAndSwapLong(Object o, long offset,Object expected, Object x);

getAndAddXX(Object var1, long var2, XX var4) 先读取一次，再读取一次判断是否修改：未修改，则将当前的值加上var4，并更新；已修改，则再次进行循环。
getAndSetXX(Object var1, long var2, XX var4) 先读取一次，再读取一次判断是否修改：未修改，则将当前的值更新为var4；已修改，则再次进行循环。
通过循环不断重试，直至CAS设置成功。

> public final int getAndAddInt(Object var1, long var2, int var4) {
> int var5;
> do {
> var5 = this.getIntVolatile(var1, var2);
> } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));
> return var5;
> }
>
> public final long getAndAddLong(Object var1, long var2, long var4) {
> long var6;
> do {
> var6 = this.getLongVolatile(var1, var2);
> } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));
> return var6;
> }
>
> public final int getAndSetInt(Object var1, long var2, int var4) {
> int var5;
> do {
> var5 = this.getIntVolatile(var1, var2);
> } while(!this.compareAndSwapInt(var1, var2, var5, var4));
> return var5;
> }
>
> public final long getAndSetLong(Object var1, long var2, long var4) {
> long var6;
> do {
> var6 = this.getLongVolatile(var1, var2);
> } while(!this.compareAndSwapLong(var1, var2, var6, var4));
> return var6;
> }
>
> public final Object getAndSetObject(Object var1, long var2, Object var4) {
> Object var5;
> do {
> var5 = this.getObjectVolatile(var1, var2);
> } while(!this.compareAndSwapObject(var1, var2, var5, var4));
> return var5;
> }

​	在较多的场景我们都可能会使用到这些原子类操作。一个典型应用就是计数了，在多线程的情况下需要考虑线程安全问题。
​	悲观锁的效率是不如乐观锁的，Atomic下的原子类的实现是类似乐观锁的，效率会比使用 synchronized 关系字高，推荐使用这种方法进行实现:

```
public class Counter {
private AtomicInteger count = new AtomicInteger();
public Counter(){}
public int getCount(){
  return count.get();
}
public void increase(){
  count.getAndIncrement();
}
}
```

CAS可能带来的问题：

​	**CAS操作可能带来ABA问题**。因为CAS操作需要在操作值的时候，检查值有没有发生变化，如果没有发发生变化则更新。如果一个值原理是A，变成了B，又变成了A，那么使用CAS进行检查时会认为它的值没有变化，但是实际上却变了。

​	ABA问题的解决办法就是使用版本号，在变量前面追加版本号，每次变量更新时把版本号加1，那么A-B-A就会变成1A-2B-3A。

​	从jdk1.5开始，jdk中的Atomic包里提供了一个类AtomicStampedReference来解决ABA问题。这个类的compareAndSet方法的作用首先检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志，如果都相等，则以原子方式将该引用和标志的值设为给定的更新值。

```
 **循环时间长开销很大。**如果CAS失败，会一直进行尝试。如果CAS长时间一直不成功，可能会给CPU带来很大的开销。 
```

​	 **只能保证一个共享变量的原子操作。**对多个共享变量操作时，循环CAS就无法保证操作的原子性。 需要对象原子化操作可以使用AtomicReference。

```
AQS(AbstractQueuedSynchronizer)，AQS是JDK下提供的一套用于实现基于FIFO等待队列的阻塞锁和相关的同步器的一个同步框架。
```

​	这个抽象类被设计为作为一些可用原子int值来表示状态的同步器的基类。例如：CountDownLatch 类的源码实现内部，有一个继承了 AbstractQueuedSynchronizer 的内部类 Sync 。 

​	AQS管理一个关于状态信息的单一整数，该整数可以表现任何状态。比如， Semaphore 用它来表现剩余的许可数，ReentrantLock 用它来表现拥有它的线程已经请求了多少次锁；FutureTask 用它来表现任务的状态(尚未开始、运行、完成和取消) 

​	使用AQS来实现一个同步器需要覆盖实现如下几个方法，并且使用 getState, setState, compareAndSetState 这几个方法来设置获取状态 ：

> 1. boolean tryAcquire(int arg) 
> 2. boolean tryRelease(int arg) 
> 3. int tryAcquireShared(int arg) 
> 4. boolean tryReleaseShared(int arg) 
> 5. boolean isHeldExclusively()

以上方法不需要全部实现，根据获取的锁的种类可以选择实现不同的方法，支持独占(排他)获取锁的同步器应该实现 tryAcquire、 tryRelease、isHeldExclusively 而支持共享获取的同步器应该实现 tryAcquireShared、tryReleaseShared、isHeldExclusively。