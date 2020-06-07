#### **类图：**

![](../../image/5bb34f0141e7174d60d9e4b2bbaf209536c.jpg)

#### **数据结构**

![](../../image/341243a4a08e560b9c8b3934b16372a4d73.jpg)

说明：

​    Sync queue，即同步队列，是双向链表，包括head结点和tail结点，head结点主要用作后续的调度。

​    Condition queue不是必须的，是一个单向链表，只有当使用Condition时，才会存在此单向链表，并且可能会有多个Condition queue。

注意：

​    AbstractQueuedLongSynchronizer 以 long 形式维护同步状态的一个 AbstractQueuedSynchronizer 版本。

​    AbstractQueuedLongSynchronizer 具有的结构、属性和方法与 AbstractQueuedSynchronizer 完全相同，但所有与状态相关的参数和结果都定义为 long 而不是 int。当创建需要 64 位状态的多级别锁和屏障等同步器时，就应当使用 AbstractQueuedLongSynchronizer。

#### **源码**：

```java
package java.util.concurrent.locks;

import sun.misc.Unsafe;

import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.concurrent.TimeUnit;

public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements java.io.Serializable {
    private static final long serialVersionUID = 7373984972572414691L;//版本序列号

    //头结点
    private transient volatile Node head;

    //尾结点
    private transient volatile Node tail;

    //状态
    private volatile int state;

    //自旋时间
    static final long spinForTimeoutThreshold = 1000L;

    //构造函数
    protected AbstractQueuedSynchronizer() {
    }

    static final class Node {
        // 模式，分为共享与独占
        // 共享模式
        static final Node SHARED = new Node();
        // 独占模式
        static final Node EXCLUSIVE = null;
        // 结点状态
        // CANCELLED，值为1，表示当前的线程被取消 (取消状态)
        // SIGNAL，值为-1，表示当前节点的后继节点包含的线程需要运行，也就是unpark (等待触发状态)
        // CONDITION，值为-2，表示当前节点在等待condition，也就是在condition队列中 (等待条件状态)
        // PROPAGATE，值为-3，表示当前场景下后续的acquireShared能够得以执行 (状态需要向后传播)
        // 值为0，表示当前节点在sync队列中，等待着获取锁
        static final int CANCELLED = 1;
        static final int SIGNAL = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;

        // 结点状态
        volatile int waitStatus;
        // 前驱结点
        volatile Node prev;
        // 后继结点
        volatile Node next;
        // 结点所对应的线程
        volatile Thread thread;
        // 下一个等待者
        Node nextWaiter;

        // 结点是否在共享模式下等待
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        // 获取前驱结点，若前驱结点为空，抛出异常
        final Node predecessor() throws NullPointerException {
            // 保存前驱结点
            Node p = prev;
            if (p == null) // 前驱结点为空，抛出异常
                throw new NullPointerException();
            else // 前驱结点不为空，返回
                return p;
        }

        // 无参构造函数
        Node() {
        }

        // 构造函数
        Node(Thread thread, Node mode) {
            this.nextWaiter = mode;
            this.thread = thread;
        }

        // 构造函数
        Node(Thread thread, int waitStatus) {
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }

    public class ConditionObject implements Condition, java.io.Serializable {
        // 版本号
        private static final long serialVersionUID = 1173984872572414699L;

        // condition队列的头结点
        private transient Node firstWaiter;

        // condition队列的尾结点
        private transient Node lastWaiter;

        // 构造函数
        public ConditionObject() {
        }

        // 添加新的waiter到wait队列
        private Node addConditionWaiter() {
            // 保存尾结点
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) { // 尾结点不为空，并且尾结点的状态不为CONDITION
                // 清除状态为CONDITION的结点
                unlinkCancelledWaiters();
                // 将最后一个结点重新赋值给t
                t = lastWaiter;
            }
            // 新建一个结点
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null) // 尾结点为空
                // 设置condition队列的头结点
                firstWaiter = node;
            else // 尾结点不为空
                // 设置为节点的nextWaiter域为node结点
                t.nextWaiter = node;
            // 更新condition队列的尾结点
            lastWaiter = node;
            return node;
        }

        private void doSignal(Node first) {
            // 循环
            do {
                if ((firstWaiter = first.nextWaiter) == null) // 该节点的nextWaiter为空
                    // 设置尾结点为空
                    lastWaiter = null;
                // 设置first结点的nextWaiter域
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                    (first = firstWaiter) != null); // 将结点从condition队列转移到sync队列失败并且condition队列中的头结点不为空，一直循环
        }

        private void doSignalAll(Node first) {
            // condition队列的头结点尾结点都设置为空
            lastWaiter = firstWaiter = null;
            // 循环
            do {
                // 获取first结点的nextWaiter域结点
                Node next = first.nextWaiter;
                // 设置first结点的nextWaiter域为空
                first.nextWaiter = null;
                // 将first结点从condition队列转移到sync队列
                transferForSignal(first);
                // 重新设置first
                first = next;
            } while (first != null);
        }

        // 从condition队列中清除状态为CANCEL的结点
        private void unlinkCancelledWaiters() {
            // 保存condition队列头结点
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) { // t不为空
                // 下一个结点
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) { // t结点的状态不为CONDTION状态
                    // 设置t节点的额nextWaiter域为空
                    t.nextWaiter = null;
                    if (trail == null) // trail为空
                        // 重新设置condition队列的头结点
                        firstWaiter = next;
                    else // trail不为空
                        // 设置trail结点的nextWaiter域为next结点
                        trail.nextWaiter = next;
                    if (next == null) // next结点为空
                        // 设置condition队列的尾结点
                        lastWaiter = trail;
                } else // t结点的状态为CONDTION状态
                    // 设置trail结点
                    trail = t;
                // 设置t结点
                t = next;
            }
        }

        // 唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。在从 await 返回之前，该线程必须重新获取锁。
        public final void signal() {
            if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
                throw new IllegalMonitorStateException();
            // 保存condition队列头结点
            Node first = firstWaiter;
            if (first != null) // 头结点不为空
                // 唤醒一个等待线程
                doSignal(first);
        }

        // 唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。在从 await 返回之前，每个线程都必须重新获取锁。
        public final void signalAll() {
            if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
                throw new IllegalMonitorStateException();
            // 保存condition队列头结点
            Node first = firstWaiter;
            if (first != null) // 头结点不为空
                // 唤醒所有等待线程
                doSignalAll(first);
        }

        // 等待，当前线程在接到信号之前一直处于等待状态，不响应中断
        public final void awaitUninterruptibly() {
            // 添加一个结点到等待队列
            Node node = addConditionWaiter();
            // 获取释放的状态
            int savedState = fullyRelease(node);
            boolean interrupted = false;
            while (!isOnSyncQueue(node)) { //
                // 阻塞当前线程
                LockSupport.park(this);
                if (Thread.interrupted()) // 当前线程被中断
                    // 设置interrupted状态
                    interrupted = true;
            }
            if (acquireQueued(node, savedState) || interrupted) //
                selfInterrupt();
        }

        private static final int REINTERRUPT = 1;

        private static final int THROW_IE = -1;

        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                    (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                    0;
        }

        private void reportInterruptAfterWait(int interruptMode)
                throws InterruptedException {
            if (interruptMode == THROW_IE)
                throw new InterruptedException();
            else if (interruptMode == REINTERRUPT)
                selfInterrupt();
        }

        // 等待，当前线程在接到信号或被中断之前一直处于等待状态
        public final void await() throws InterruptedException {
            if (Thread.interrupted()) // 当前线程被中断，抛出异常
                throw new InterruptedException();
            // 在wait队列上添加一个结点
            Node node = addConditionWaiter();
            //
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                // 阻塞当前线程
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) // 检查结点等待时的中断类型
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

        // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态
        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() + nanosTimeout;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return deadline - System.nanoTime();
        }

        // 等待，当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态
        public final boolean awaitUntil(Date deadline)
                throws InterruptedException {
            long abstime = deadline.getTime();
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (System.currentTimeMillis() > abstime) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                LockSupport.parkUntil(this, abstime);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。此方法在行为上等效于：awaitNanos(unit.toNanos(time)) > 0
        public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {
            long nanosTimeout = unit.toNanos(time);
            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            final long deadline = System.nanoTime() + nanosTimeout;
            boolean timedout = false;
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                if (nanosTimeout <= 0L) {
                    timedout = transferAfterCancelledWait(node);
                    break;
                }
                if (nanosTimeout >= spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
                nanosTimeout = deadline - System.nanoTime();
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null)
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
            return !timedout;
        }

        final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
            return sync == AbstractQueuedSynchronizer.this;
        }

        //  查询是否有正在等待此条件的任何线程
        protected final boolean hasWaiters() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    return true;
            }
            return false;
        }

        // 返回正在等待此条件的线程数估计值
        protected final int getWaitQueueLength() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            int n = 0;
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION)
                    ++n;
            }
            return n;
        }

        // 返回包含那些可能正在等待此条件的线程集合
        protected final Collection<Thread> getWaitingThreads() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            ArrayList<Thread> list = new ArrayList<Thread>();
            for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
                if (w.waitStatus == Node.CONDITION) {
                    Thread t = w.thread;
                    if (t != null)
                        list.add(t);
                }
            }
            return list;
        }
    }

    //返回同步状态的当前值
    protected final int getState() {
        return state;
    }

    //设置同步状态的值
    protected final void setState(int newState) {
        state = newState;
    }

    //如果当前状态值等于预期值expect，则以原子方式将同步状态设置为给定的更新值update
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

    // 入队列
    private Node enq(final Node node) {
        for (; ; ) { // 无限循环，确保结点能够成功入队列
            // 保存尾结点
            Node t = tail;
            if (t == null) { // 尾结点为空，即还没被初始化
                if (compareAndSetHead(new Node())) // 头结点为空，并设置头结点为新生成的结点
                    tail = head; // 头结点与尾结点都指向同一个新生结点
            } else { // 尾结点不为空，即已经被初始化过
                // 将node结点的prev域连接到尾结点
                node.prev = t;
                if (compareAndSetTail(t, node)) { // 比较结点t是否为尾结点，若是则将尾结点设置为node
                    // 设置尾结点的next域为node
                    t.next = node;
                    return t; // 返回node节点的前一个节点
                }
            }
        }
    }

    // 添加一个节点到队列的尾部，并返回这个节点
    private Node addWaiter(Node mode) {
        // 新生成一个结点，默认为独占模式
        Node node = new Node(Thread.currentThread(), mode);
        // 保存尾结点
        Node pred = tail;
        if (pred != null) { // 尾结点不为空，即已经被初始化
            // 将node结点的prev域连接到尾结点
            node.prev = pred;
            if (compareAndSetTail(pred, node)) { // 比较pred是否为尾结点，是则将尾结点设置为node
                // 设置尾结点的next域为node
                pred.next = node;
                return node; // 返回新生成的结点
            }
        }
        enq(node); // 尾结点为空(即还没有被初始化过)，或者是compareAndSetTail操作失败，则入队列
        return node;//返回这个节点
    }


    private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
    }

    //  唤醒node的后继不为CANCELLED状态的节点
    private void unparkSuccessor(Node node) {
        // 获取node结点的等待状态
        int ws = node.waitStatus;
        if (ws < 0) // 状态值小于0，为SIGNAL -1 或 CONDITION -2 或 PROPAGATE -3
            // 比较并且设置结点等待状态，设置为0
            compareAndSetWaitStatus(node, ws, 0);

        // 获取node节点的下一个结点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) { // 下一个结点为空或者下一个节点的等待状态大于0，即为CANCELLED
            // s赋值为空
            s = null;
            // 从尾结点开始从后往前开始遍历
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0) // 找到等待状态小于等于0的结点，找到最前的状态小于等于0的结点
                    // 保存结点
                    s = t;
        }
        if (s != null) // 该结点不为为空，释放许可
            LockSupport.unpark(s.thread);
    }

    private void doReleaseShared() {
        for (; ; ) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;
                    unparkSuccessor(h);
                } else if (ws == 0 &&
                        !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;
            }
            if (h == head)
                break;
        }
    }


    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head;
        setHead(node);

        if (propagate > 0 || h == null || h.waitStatus < 0 ||
                (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }

    // 取消继续获取(资源)，将当前节点等待状态置为CANCELLED
    private void cancelAcquire(Node node) {
        // node为空，返回
        if (node == null)
            return;
        // 设置node结点的thread为空
        node.thread = null;
        // 保存node的前驱结点
        Node pred = node.prev;
        while (pred.waitStatus > 0) // 找到node前驱结点中第一个状态小于0的结点，即不为CANCELLED状态的结点
            node.prev = pred = pred.prev;
        // 获取pred结点的下一个结点
        Node predNext = pred.next;
        // 设置node结点的状态为CANCELLED
        node.waitStatus = Node.CANCELLED;

        //此时node.prev指向了前驱节点中第一个不为CANCELLED的节点
        //pred则指向了那个节点，predNext指向了那个节点的下一个节点

        if (node == tail && compareAndSetTail(node, pred)) { // node结点为尾结点，则设置尾结点为pred结点
            // 比较并设置predNext节点为null
            compareAndSetNext(pred, predNext, null);
        } else { // node结点不为尾结点，或者比较设置不成功
            int ws;
            //（pred结点不为头结点，并且pred结点的状态为SIGNAL）或者
            // pred结点状态小于等于0(不为CANCELLED)，并且比较并设置pred等待状态为SIGNAL成功，并且pred结点所封装的线程不为空
            if (pred != head && ((ws = pred.waitStatus) == Node.SIGNAL || (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) && pred.thread != null) {
                Node next = node.next;//next指向node的后继节点
                if (next != null && next.waitStatus <= 0) // 若 next 不空并且状态不为CANCELLED
                    compareAndSetNext(pred, predNext, next); // 比较并设置pred.next = next;

                //此时pred.next指向了next，next.prev指向了pred
            } else {//当前节点为头节点
                unparkSuccessor(node); // 唤醒node的后继不为CANCELLED状态的节点
            }
            node.next = node; // help GC
        }
    }

    // 当获取(资源)失败后，检查并且更新结点状态
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        // 获取前驱结点的状态
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL) // 状态为SIGNAL，为-1
            // 可以进行park操作
            return true;
        if (ws > 0) { // 表示状态为CANCELLED，为1
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0); // 找到pred结点前面最近的一个状态不为CANCELLED的结点
            // 赋值pred结点的next域
            pred.next = node;
        } else { // 为PROPAGATE -3 或者是0 表示无状态,(为CONDITION -2时，表示此节点在condition queue中)
            // 比较并设置前驱结点的状态为SIGNAL
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        // 不能进行park操作
        return false;
    }

    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }

    // 进行park操作并且返回该线程是否被中断
    private final boolean parkAndCheckInterrupt() {
        // 在许可可用之前禁用当前线程，并且设置了blocker
        LockSupport.park(this);
        return Thread.interrupted(); // 当前线程是否已被中断，并清除中断标记位
    }

    // sync队列中的结点在独占且忽略中断的模式下获取(资源)
    final boolean acquireQueued(final Node node, int arg) {
        // 标志
        boolean failed = true;
        try {
            // 中断标志
            boolean interrupted = false;
            for (; ; ) { // 无限循环
                // 获取node节点的前驱结点
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) { // 前驱为头结点并且成功获得锁
                    setHead(node); // 设置头结点
                    p.next = null; // help GC
                    failed = false; // 设置标志
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private void doAcquireInterruptibly(int arg)
            throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (; ; ) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (; ; ) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                        nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private void doAcquireShared(int arg) {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (; ; ) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private void doAcquireSharedInterruptibly(int arg)
            throws InterruptedException {
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (; ; ) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                        parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            for (; ; ) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        failed = false;
                        return true;
                    }
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                        nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

    // 试图在独占模式下获取对象状态
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

    //试图设置状态来反映独占模式下的一个释放
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }

    //试图在共享模式下获取对象状态
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    //试图设置状态来反映共享模式下的一个释放
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

    //如果对于当前（正调用的）线程，同步是以独占方式进行的，则返回 true
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }

    //以独占模式获取对象，忽略中断
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
                acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

    //以独占模式获取对象，如果被中断则中止
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }

    //试图以独占模式获取对象，如果被中断则中止，如果到了给定超时时间，则会失败
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
                doAcquireNanos(arg, nanosTimeout);
    }

    //以独占模式释放对象
    public final boolean release(int arg) {
        if (tryRelease(arg)) { // 释放成功
            // 保存头结点
            Node h = head;
            if (h != null && h.waitStatus != 0) // 头结点不为空并且头结点状态不为0
                unparkSuccessor(h); //释放头结点的后继结点
            return true;
        }
        return false;
    }

    //以共享模式获取对象，忽略中断
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    //以共享模式获取对象，如果被中断则中止
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }

    //试图以共享模式获取对象，如果被中断则中止，如果到了给定超时时间，则会失败
    public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquireShared(arg) >= 0 ||
                doAcquireSharedNanos(arg, nanosTimeout);
    }

    //以共享模式释放对象
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

    //查询是否有正在等待获取的任何线程
    public final boolean hasQueuedThreads() {
        return head != tail;
    }

    //查询是否其他线程也曾争着获取此同步器；也就是说，是否某个 acquire 方法已经阻塞
    public final boolean hasContended() {
        return head != null;
    }

    //返回队列中第一个（等待时间最长的）线程，如果目前没有将任何线程加入队列，则返回 null.
    // 在此实现中，该操作是以固定时间返回的，但是，如果其他线程目前正在并发修改该队列，则可能出现循环争用
    public final Thread getFirstQueuedThread() {
        // handle only fast path, else relay
        return (head == tail) ? null : fullGetFirstQueuedThread();
    }

    private Thread fullGetFirstQueuedThread() {
        Node h, s;
        Thread st;
        if (((h = head) != null && (s = h.next) != null &&
                s.prev == head && (st = s.thread) != null) ||
                ((h = head) != null && (s = h.next) != null &&
                        s.prev == head && (st = s.thread) != null))
            return st;
        Node t = tail;
        Thread firstThread = null;
        while (t != null && t != head) {
            Thread tt = t.thread;
            if (tt != null)
                firstThread = tt;
            t = t.prev;
        }
        return firstThread;
    }

    //如果给定线程的当前已加入队列，则返回 true
    public final boolean isQueued(Thread thread) {
        if (thread == null)
            throw new NullPointerException();
        for (Node p = tail; p != null; p = p.prev)
            if (p.thread == thread)
                return true;
        return false;
    }

    final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
                (s = h.next) != null &&
                !s.isShared() &&
                s.thread != null;
    }
    
    //判断是否有其他线程等待获取的时间比当前线程更长
    public final boolean hasQueuedPredecessors() {
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
                ((s = h.next) == null || s.thread != Thread.currentThread());
    }

    //返回等待获取的线程数估计值
    public final int getQueueLength() {
        int n = 0;
        for (Node p = tail; p != null; p = p.prev) {
            if (p.thread != null)
                ++n;
        }
        return n;
    }

    // 返回包含可能正在等待获取的线程 collection
    public final Collection<Thread> getQueuedThreads() {
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node p = tail; p != null; p = p.prev) {
            Thread t = p.thread;
            if (t != null)
                list.add(t);
        }
        return list;
    }

    //返回包含可能正以独占模式等待获取的线程 collection
    public final Collection<Thread> getExclusiveQueuedThreads() {
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node p = tail; p != null; p = p.prev) {
            if (!p.isShared()) {
                Thread t = p.thread;
                if (t != null)
                    list.add(t);
            }
        }
        return list;
    }

    //返回包含可能正以共享模式等待获取的线程 collection
    public final Collection<Thread> getSharedQueuedThreads() {
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node p = tail; p != null; p = p.prev) {
            if (p.isShared()) {
                Thread t = p.thread;
                if (t != null)
                    list.add(t);
            }
        }
        return list;
    }

    //返回标识此同步器及其状态的字符串
    public String toString() {
        int s = getState();
        String q = hasQueuedThreads() ? "non" : "";
        return super.toString() +
                "[State = " + s + ", " + q + "empty queue]";
    }

    final boolean isOnSyncQueue(Node node) {
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;

        return findNodeFromTail(node);
    }

    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (; ; ) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }

    final boolean transferForSignal(Node node) {
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        Node p = enq(node);
        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }


    final boolean transferAfterCancelledWait(Node node) {
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);
            return true;
        }
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }

    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }

    //查询给定的 ConditionObject 是否使用了此同步器作为其锁
    public final boolean owns(ConditionObject condition) {
        return condition.isOwnedBy(this);
    }

    //查询是否有线程正在等待给定的、与此同步器相关的条件
    public final boolean hasWaiters(ConditionObject condition) {
        if (!owns(condition))
            throw new IllegalArgumentException("Not owner");
        return condition.hasWaiters();
    }

    public final int getWaitQueueLength(ConditionObject condition) {
        if (!owns(condition))
            throw new IllegalArgumentException("Not owner");
        return condition.getWaitQueueLength();
    }

    //返回一个 collection，其中包含可能正在等待与此同步器有关的给定条件的那些线程
    public final Collection<Thread> getWaitingThreads(ConditionObject condition) {
        if (!owns(condition))
            throw new IllegalArgumentException("Not owner");
        return condition.getWaitingThreads();
    }

    //Unsafe类实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //state内存偏移地址
    private static final long stateOffset;
    //head内存偏移地址
    private static final long headOffset;
    //state内存偏移地址
    private static final long tailOffset;
    //tail内存偏移地址
    private static final long waitStatusOffset;
    //next内存偏移地址
    private static final long nextOffset;

    //静态初始化块
    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                    (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                    (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                    (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                    (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                    (Node.class.getDeclaredField("next"));

        } catch (Exception ex) {
            throw new Error(ex);
        }
    }


    private final boolean compareAndSetHead(Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }


    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }


    private static final boolean compareAndSetWaitStatus(Node node, int expect, int update) {
        return unsafe.compareAndSwapInt(node, waitStatusOffset,
                expect, update);
    }


    private static final boolean compareAndSetNext(Node node, Node expect, Node update) {
        return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
    }
}
```

#### 类 AbstractQueuedSynchronizer

**所有已实现的接口：**

[Serializable](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/io/Serializable.html)

为实现依赖于先进先出 (FIFO) 等待队列的阻塞锁和相关同步器（信号量、事件，等等）提供一个框架。

 此类支持默认的 独占 模式和 共享 模式之一，或者二者都支持。

-   处于独占模式下时，其他线程试图获取该锁将无法取得成功。
-   在共享模式下，多个线程获取某个锁可能（但不是一定）会获得成功。

    只支持独占模式或者只支持共享模式的子类不必定义支持未使用模式的方法。

#### **使用**

在ReentrantLock，Semaphore，CountDownLatch，ReentrantReadWriteLock中都用到了继承自AQS的Sync内部类。

AQS根据模式的不同：独占（EXCLUSIVE）和共享（SHARED）模式。

-   独占：只有一个线程能执行。如ReentrantLock。
-   共享：多个线程可同时执行。如Semaphore，可以设置指定数量的线程共享资源。

    为了将此类用作同步器的基础，需要适当地重新定义以下方法。实现以下方法，主要是通过使用 [`getState()`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/concurrent/locks/AbstractQueuedSynchronizer.html#getState())、[`setState(int)`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/concurrent/locks/AbstractQueuedSynchronizer.html#setState(int)) 、[`compareAndSetState(int, int)`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/concurrent/locks/AbstractQueuedSynchronizer.html#compareAndSetState(int,%20int)) 方法来检查、修改同步状态来实现。

-   [`tryAcquire(int)`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/concurrent/locks/AbstractQueuedSynchronizer.html#tryAcquire(int))//独占模式下获取锁的方法
-   [`tryRelease(int)`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/concurrent/locks/AbstractQueuedSynchronizer.html#tryRelease(int))//独占模式下解锁的方法
-   [`tryAcquireShared(int)`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/concurrent/locks/AbstractQueuedSynchronizer.html#tryAcquireShared(int))//共享模式下获取锁的方法
-   [`tryReleaseShared(int)`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/concurrent/locks/AbstractQueuedSynchronizer.html#tryReleaseShared(int))//共享模式下解锁的方法
-   [`isHeldExclusively()`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/concurrent/locks/AbstractQueuedSynchronizer.html#isHeldExclusively())//判断是否为持有独占锁

    默认情况下，每个方法都抛出 [`UnsupportedOperationException`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/lang/UnsupportedOperationException.html)。这些方法的实现，在内部必须是线程安全的，通常应该很短并且不被阻塞。其他所有方法都被声明为final，因为它们无法保证是各不相同的。

#### 示例:

对应的类根据不同的模式，来**实现**对应的方法。例如：独占锁只需实现`[tryAcquire(int)]、[tryRelease(int)]、`[`isHeldExclusively*()`]，而不必要实现``[tryAcquireShared(int)]、[tryReleaseShared(int)]。

以下是一个非重入的互斥锁类，它使用值 0 表示未锁定状态，使用 1 表示锁定状态。

```java
 class Mutex implements Lock, java.io.Serializable {

    // Our internal helper class
    private static class Sync extends AbstractQueuedSynchronizer {
      // Report whether in locked state
      protected boolean isHeldExclusively() { 
        return getState() == 1; 
      }

      // Acquire the lock if state is zero
      public boolean tryAcquire(int acquires) {
        assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
      }

      // Release the lock by setting state to zero
      protected boolean tryRelease(int releases) {
        assert releases == 1; // Otherwise unused
        if (getState() == 0) throw new IllegalMonitorStateException();
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
      }
       
      // Provide a Condition
      Condition newCondition() { return new ConditionObject(); }

      // Deserialize properly
      private void readObject(ObjectInputStream s) 
        throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
      }
    }

    // The sync object does all the hard work. We just forward to it.
    private final Sync sync = new Sync();

    public void lock()                { sync.acquire(1); }
    public boolean tryLock()          { return sync.tryAcquire(1); }
    public void unlock()              { sync.release(1); }
    public Condition newCondition()   { return sync.newCondition(); }
    public boolean isLocked()         { return sync.isHeldExclusively(); }
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    public void lockInterruptibly() throws InterruptedException { 
      sync.acquireInterruptibly(1);
    }
    public boolean tryLock(long timeout, TimeUnit unit) 
        throws InterruptedException {
      return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
 }
```

以下是一个锁存器类，它类似于 [`CountDownLatch`](http://tool.oschina.net/uploads/apidocs/jdk-zh/java/util/concurrent/CountDownLatch.html)，除了只需要触发单个 signal 之外。因为锁存器是非独占的，所以它使用 shared 的获取和释放方法。

```java
class BooleanLatch {

    private static class Sync extends AbstractQueuedSynchronizer {
      boolean isSignalled() { return getState() != 0; }

      protected int tryAcquireShared(int ignore) {
        return isSignalled()? 1 : -1;
      }
        
      protected boolean tryReleaseShared(int ignore) {
        setState(1);
        return true;
      }
    }

    private final Sync sync = new Sync();
    public boolean isSignalled() { return sync.isSignalled(); }
    public void signal()         { sync.releaseShared(1); }
    public void await() throws InterruptedException {
      sync.acquireSharedInterruptibly(1);
    }
 }
```
