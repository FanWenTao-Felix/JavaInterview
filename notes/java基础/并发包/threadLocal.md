### Threadlocal原理

`ThreadLocal` 为解决多线程程序的并发问题提供了一种新的思路。使用这个工具类可以很简洁地编写出优美的多线程程序。当使用 ThreadLocal 维护变量时，ThreadLocal 为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。

每个线程中都保有一个`ThreadLocalMap`的成员变量，`ThreadLocalMap `内部采用`WeakReference`数组保存，数组的key即为`ThreadLocal `内部的Hash值。

![img](../../image/2020-02-13-19-17-12.png)

## 内存泄漏

`ThreadLocalMap` 使用 `ThreadLocal` 的弱引用作为 key ，如果一个 `ThreadLocal` 没有外部强引用来引用它，那么系统 GC 的时候，这个 `ThreadLocal` 势必会被回收，这样一来，`ThreadLocalMap` 中就会出现 `key` 为 `null` 的 `Entry` ，就没有办法访问这些 `key` 为 `null` 的 `Entry` 的 `value`，如果当前线程再迟迟不结束的话，这些 `key` 为 `null` 的 `Entry` 的 `value` 就会一直存在一条强引用链：`Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value` 永远无法回收，造成内存泄漏。

```
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

其实，`ThreadLocalMap` 的设计中已经考虑到这种情况，也加上了一些防护措施：在 `ThreadLocal` 的 `get(),set(),remove()`的时候都会清除线程 `ThreadLocalMap` 里所有 `key` 为 `null` 的 `value`



### ThreadLocal原理，注意事项，参数传递

>  线程本地变量，线程独有的变量，作用域为当前线程 
>
>  **ThreadLocal类源码分析**
>
>  set方法
>
>  ​	先来看set方法，源码如下：
>
>  ```csharp
>  public void set(T value) {
>     Thread t = Thread.currentThread();
>     ThreadLocalMap map = getMap(t);
>     if (map != null)
>         map.set(this, value);
>     else
>         createMap(t, value);
>  }
>  
>  void createMap(Thread t, T firstValue) {
>     t.threadLocals = new ThreadLocalMap(this, firstValue);
>  }
>  
>  ThreadLocalMap getMap(Thread t) {
>     return t.threadLocals;
>  }
>  ```
>
>  ​	set方法是通过一个ThreadLocalMap对象来set值，而ThreadLocalMap虽然是ThreadLocal的静态内部类，却是Thread类中的成员变量。
>
>  ​	get方法
>
>  ```java
>  public T get() {
>     Thread t = Thread.currentThread();
>     ThreadLocalMap map = getMap(t);
>     if (map != null) {
>         ThreadLocalMap.Entry e = map.getEntry(this);
>         if (e != null) {
>             @SuppressWarnings("unchecked")
>             T result = (T)e.value;
>             return result;
>         }
>     }
>     return setInitialValue();
>  }
>  
>  private T setInitialValue() {
>     T value = initialValue();
>     Thread t = Thread.currentThread();
>     ThreadLocalMap map = getMap(t);
>     if (map != null)
>         map.set(this, value);
>     else
>         createMap(t, value);
>     return value;
>  }
>  
>  protected T initialValue() {
>     return null;
>  }
>  ```
>
>  ​	get方法将操作委托给了ThreadLocalMap，通过最终获得ThreadLocalMap.Entry来获取最终的值。
>
>  ### remove方法
>
>  ​	remove方法同样是交给ThreadLocalMap来处理
>
>  ```csharp
>  public void remove() {
>      ThreadLocalMap m = getMap(Thread.currentThread());
>      if (m != null)
>          m.remove(this);
>  }
>  ```
>
>  **ThreadLocalMap源码分析**
>
>  ​	ThreadLocalMap是线程的私有成员变量。这样做是为了避免多线程竞争，此时是线程私有，处理的时候不需要加锁。由于ThreadLocal本身的设计就是变量不与其他线程共享，不需要其他线程访问本对象的变量，放在Thread对象中不会有问题。
>
>  ​	Entry数据结构
>
>  ​	ThreadLocalMap维护了一个Entry类型的数据结构：
>
>  ```dart
>     static class Entry extends WeakReference<ThreadLocal<?>> {
>         /** The value associated with this ThreadLocal. */
>         Object value;
>         Entry(ThreadLocal<?> k, Object v) {
>             super(k);
>             value = v;
>         }
>     }
>     private Entry[] table;
>  ```
>
>  ​	Entry是一个以ThreadLocal为key，Object为value的键值对。需要注意的是，threadLocal是弱引用，即使线程正在执行中，只要ThreadLocal对象引用被置成null, Entry的Key就会自动在下一次YGC时被垃圾回收。而在 ThreadLocal使用set()和get()时，又会自动地将那些 key==null 的value置为null，使value能够被垃圾回收，避免内存泄漏。 ThreadLocal也因为这样的设计导致了一些问题。
>
>  ### set方法
>
>  ```csharp
>  private void set(ThreadLocal<?> key, Object value) { 
>     Entry[] tab = table;
>     int len = tab.length;
>     // 定位Entry存放的位置
>     int i = key.threadLocalHashCode & (len-1);
>     // 处理hash冲突的情况，这里采用的是开放地址法
>     for (Entry e = tab[i];
>          e != null;
>          e = tab[i = nextIndex(i, len)]) {
>         ThreadLocal<?> k = e.get();
>         //更新
>         if (k == key) {
>             e.value = value;
>             return;
>         }
>         if (k == null) {
>             replaceStaleEntry(key, value, i);
>             return;
>         }
>     }
>     // 新建entry并插入
>     tab[i] = new Entry(key, value);
>     int sz = ++size;
>     // 清除脏数据，扩容
>     if (!cleanSomeSlots(i, sz) && sz >= threshold)
>         rehash();
>  }
>  ```
>
>  ### get方法
>
>  ```csharp
>  private Entry getEntry(ThreadLocal<?> key) {
>     // 确定entry位置
>     int i = key.threadLocalHashCode & (table.length - 1);
>     Entry e = table[i];
>     // 命中
>     if (e != null && e.get() == key)
>         return e;
>     else
>         // 存在hash冲突，继续查找
>         return getEntryAfterMiss(key, i, e);
>  }
>  
>  private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
>     Entry[] tab = table;
>     int len = tab.length;
>  
>     while (e != null) {
>         ThreadLocal<?> k = e.get();
>         //找到entry
>         if (k == key)
>             return e;
>         // 脏数据处理
>         if (k == null)
>             expungeStaleEntry(i);
>         else
>             //遍历
>             i = nextIndex(i, len);
>         e = tab[i];
>     }
>     return null;
>  }
>  ```
>
>  ### remove方法
>
>  ```csharp
>  private void remove(ThreadLocal<?> key) {
>     Entry[] tab = table;
>     int len = tab.length;
>     int i = key.threadLocalHashCode & (len-1);
>     for (Entry e = tab[i];
>          e != null;
>          e = tab[i = nextIndex(i, len)]) {
>         if (e.get() == key) {
>             //清空key
>             e.clear();
>             //清空value
>             expungeStaleEntry(i);
>             return;
>         }
>     }
>  }
>  ```
>
>  - 1个ThreadLocal对象可以被多个线程所共享
>  - 1个Thread有且仅有1个ThreadLocalMap对象
>  - 1个ThreadLocalMap对象存储多个Entry对象
>  - ThreadLocal对象不持有Value，Value由线程的Entry对象持有
>  - 1个Entry对象的key弱引用指向1个ThreadLocal对象
>
>  **注意事项**
>
>  脏数据
>
>  ​	线程复用会产生脏数据。由于线程池会重用Thread对象，那么如果在实现的线程run()方法体中不显式地调用remove() 清理与线程相关的ThreadLocalMap 信息，那么倘若下一个线程不调用set() 设置初始值，就可能get() 到重用的线程信息，包括 ThreadLocal所关联的线程对象的value值。
>
>  内存泄漏
>
>  ​	通常我们会使用使用static关键字来修饰ThreadLocal（1个ThreadLocal 实例可以被多个线程共享）。在此场景下，其生命周期就不会随着线程结束而结束，寄希望于ThreadLocal对象失去引用后，触发弱引用机制来回收Entry的Value就不现实了。如果不进行remove() 操作，那么这个线程执行完成后，通过ThreadLocal对象持有的对象是不会被释放的。
>
>  ​	以上两个问题的解决办法很简单，就是在每次用完ThreadLocal时， 必须要及时调用 remove()方法清理。
>
>  父子线程共享线程变量
>
>  ​	很多场景下通过ThreadLocal来透传全局上下文，会发现子线程的value和主线程不一致。比如用ThreadLocal来存储监控系统的某个标记位，暂且命名为traceId。某次请求下所有的traceld都是一致的，以获得可以统一解析的日志文件。但在实际开发过程中，发现子线程里的traceld为null，跟主线程的并不一致。这就需要使用InheritableThreadLocal来解决父子线程之间共享线程变量的问题，使整个连接过程中的traceId一致。