可以通过Executors静态工厂构建线程池，但一般不建议这样使用

因为这种创建线程池的方式有很大的隐患，稍有不慎就有可能导致线上故障。

# Executors

    Executors 是一个Java中的工具类。提供工厂方法来创建不同类型的线程池。

![](../image/up-888b0e3e0c78f098a14b7011df826b00071.png)

    Executors的创建线程池的方法，创建出来的线程池都实现了ExecutorService接口。

常用方法有以下几个：

newFiexedThreadPool(int Threads)：创建固定数目线程的线程池。

newCachedThreadPool()：创建一个可缓存的线程池，调用execute 将重用以前构造的线程（如果线程可用）。如果没有可用的线程，则创建一个新线程并添加到池中。终止并从缓存中移除那些已有 60 秒钟未被使用的线程。

newSingleThreadExecutor()创建一个单线程化的Executor。

newScheduledThreadPool(int corePoolSize)创建一个支持定时及周期性的任务执行的线程池，多数情况下可用来替代Timer类。

Executors的功能还是比较强大的，又用到了工厂模式、又有比较强的扩展性，重要的是用起来还比较方便，如：

>     ExecutorService executor = Executors.newFixedThreadPool(nThreads) ;

即可创建一个固定大小的线程池。

# Executors存在什么问题

在阿里巴巴Java开发手册中提到，使用Executors创建线程池可能会导致OOM(OutOfMemory ,内存溢出)，但是并没有说明为什么，那么接下来我们就来看一下到底为什么不允许使用Executors？

**    一个简单的例子，模拟一下使用Executors导致OOM的情况。**

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ExecutorsDemo {
    private static ExecutorService executor = Executors.newFixedThreadPool(15);

    public static void main(String[] args) {
        for (int i = 0; i < Integer.MAX_VALUE; i++) {
            executor.execute(new SubThread());
        }
    }
}

class SubThread implements Runnable {
    @Override
    public void run() {
        try {
            Thread.sleep(10000);
        } catch (InterruptedException e) {
            //do nothing
        }
    }
}
```

**通过指定JVM参数：-Xmx8m -Xms8m 运行以上代码，会抛出OOM:**

![](../image/up-3fac313e18f9225734c23e96397a5480eb7.png)

以上代码指出，ExecutorsDemo.java的第16行，就是代码中的executor.execute(new SubThread());。

# Executors为什么存在缺陷

通过上面的例子，我们知道了Executors创建的线程池存在OOM的风险，那么到底是什么原因导致的呢？我们需要深入Executors的源码来分析一下。

其实，在上面的报错信息中，我们是可以看出蛛丝马迹的，在以上的代码中其实已经说了，真正的导致OOM的其实是LinkedBlockingQueue.offer方法。

![image-20200607160513353](../image/image-20200607160513353.png)

看代码的话，也可以发现，其实底层确实是通过LinkedBlockingQueue实现的：

```java
public static ExecutorService newFixedThreadPool(int nThreads){
   return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}
```

如果对Java中的阻塞队列有所了解的话，看到这里或许就能够明白原因了。

Java中的BlockingQueue主要有两种实现，分别是ArrayBlockingQueue 和 LinkedBlockingQueue。

-       ArrayBlockingQueue是一个用数组实现的有界阻塞队列，必须设置容量。
-       LinkedBlockingQueue是一个用链表实现的有界阻塞队列，容量可以选择进行设置，不设置的话，将是一个无边界的阻塞队列，最大长度为Integer.MAX_VALUE。

    问题就出在：不设置的话，将是一个无边界的阻塞队列，最大长度为Integer.MAX\_VALUE。也就是说，如果我们不设置LinkedBlockingQueue的容量的话，其默认容量将会是Integer.MAX\_VALUE。

    而newFixedThreadPool中创建LinkedBlockingQueue时，并未指定容量。此时，LinkedBlockingQueue就是一个无边界队列，对于一个无边界队列来说，是可以不断的向队列中加入任务的，这种情况下就有可能因为任务过多而导致内存溢出问题。

    上面提到的问题主要体现在newFixedThreadPool和newSingleThreadExecutor两个工厂方法上，并不是说newCachedThreadPool和newScheduledThreadPool这两个方法就安全了，这两种方式创建的最大线程数可能是Integer.MAX_VALUE，而创建这么多线程，必然就有可能导致OOM。

# 创建线程池的正确姿势

避免使用Executors创建线程池，主要是避免使用其中的默认实现，那么我们可以自己直接调用ThreadPoolExecutor的构造函数来自己创建线程池。在创建的同时，给BlockQueue指定容量就可以了。

```java
private static ExecutorService executor = new ThreadPoolExecutor(10, 10, 60L, TimeUnit.SECONDS, new ArrayBlockingQueue(10));
```

这种情况下，一旦提交的线程数超过当前可用线程数时，就会抛出java.util.concurrent.RejectedExecutionException，这是因为当前线程池使用的队列是有边界队列，队列已经满了便无法继续处理新的请求。但是异常（Exception）总比发生错误（Error）要好。

除了自己定义ThreadPoolExecutor外。还有其他开源类库，如apache和guava等。推荐使用guava提供的ThreadFactoryBuilder来创建线程池。

```java
    public class ExecutorsDemo {
        private static ThreadFactory namedThreadFactory = new ThreadFactoryBuilder().setNameFormat("demo-pool-%d").build();
        private static ExecutorService pool = new ThreadPoolExecutor(5, 200, 0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>(1024), namedThreadFactory,
                new ThreadPoolExecutor.AbortPolicy());

        public static void main(String[] args) {
            for (int i = 0; i < Integer.MAX_VALUE; i++) {
                pool.execute(new SubThread());
            }
        }
    }
```

通过上述方式创建线程时，不仅可以避免OOM的问题，还可以自定义线程名称，更加方便的出错的时候溯源。