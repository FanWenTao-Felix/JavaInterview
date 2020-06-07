针对HotSpot VM的实现，它里面的GC其实准确分类只有两大种：

-   Partial GC：并不收集整个GC堆的模式
    -   Young GC：只收集young gen的GC
    -   Old GC：只收集old gen的GC。只有CMS的concurrent collection是这个模式
    -   Mixed GC：收集整个young gen以及部分old gen的GC。只有G1有这个模式
-   Full GC：收集整个堆，包括young gen、old gen、perm gen（如果存在的话）等所有部分的模式。

    Major GC通常是跟full GC是等价的，收集整个GC堆。但因为HotSpot VM发展了这么多年，外界对各种名词的解读已经完全混乱了，当有人说“major GC”的时候一定要问清楚他想要指的是上面的full GC还是old GC。

    最简单的分代式GC策略，按HotSpot VM的serial GC的实现来看，触发条件是：

-   young GC：当young gen中的eden区分配满的时候触发。
    
    -   注意young GC中有部分存活对象会晋升到old gen，所以young GC后old gen的占用量通常会有所升高。
-   full GC：当准备要触发一次young GC时，如果发现统计数据之前 young GC的平均晋升大小 比 目前old gen剩余的空间大，则不会触发young GC，而是转为触发full GC（因为HotSpot VM的GC里，除了 CMS 的 concurrent collection之外，其它能收集 old gen 的GC都会同时收集整个GC堆，包括 young gen，所以不需要事先触发一次单独的young GC）
    -   如果有perm gen的话，要在perm gen分配空间但已经没有足够空间时，也要触发一次full GC；或者System.gc()、heap dump带GC，默认也是触发full GC。

    HotSpot VM里其它非并发GC的触发条件复杂一些，不过大致的原理与上面说的其实一样。

    当然也总有例外。Parallel Scavenge（-XX:+UseParallelGC）框架下，默认是在要触发full GC前先执行一次young GC，并且两次GC之间能让应用程序稍微运行一小下，以期降低full GC的暂停时间（因为young GC会尽量清理了young gen的死对象，减少了full GC的工作量）。控制这个行为的VM参数是-XX:+ScavengeBeforeFullGC。

     并发GC的触发条件就不太一样。以CMS GC为例，它主要是定时去检查old gen的使用量，当使用量超过了触发比例就会启动一次CMS GC，对old gen做并发收集。

    Hotspot JVM实现中几种GC算法：

    1 Serial GC算法：Serial Young GC ＋ Serial Old GC （实际上它是全局范围的Full GC）；

    2 Parallel GC算法：Parallel Young GC ＋ 非并行的PS MarkSweep GC / 并行的Parallel Old GC（实际上也是全局范围的Full GC）

        选PS MarkSweep GC 还是 Parallel Old GC 由参数UseParallelOldGC来控制；

    3 CMS算法：ParNew（Young）GC + CMS（Old）GC ＋ Full GC for CMS算法；

    4 G1 GC：Young GC + mixed GC（新生代，再加上部分老生代）＋ Full GC for G1 GC算法