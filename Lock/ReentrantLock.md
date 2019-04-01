## ReentrantLock
#### 实现了Lock和Serializable接口
#### Sync类继承了AbstractQueuedSynchronizer
#### 默认非公平锁
#### 支持可重入
#### 公平锁的tryAcquire和非公平锁的tryAcquire基本一致，只是再获取锁时先判断当前节点是否有前驱节点
#### 阻塞队列不包含head，延迟初始化head，也就是只有第二个线程有冲突，才会初始化head
#### 非公平锁在调用 lock 后，首先就会调用 CAS 进行一次抢锁，如果这个时候恰巧锁没有被占用，那么直接就获取到锁返回了。
#### 非公平锁在 CAS 失败后，和公平锁一样都会进入到 tryAcquire 方法，在 tryAcquire 方法中，如果发现锁这个时候被释放了（state == 0），非公平锁会直接 CAS 抢锁，但是公平锁会判断等待队列是否有线程处于等待状态，如果有则不去抢锁，乖乖排到后面。
#### 即使发生了中断，节点依然会转移到阻塞队列。
```
package java.util.concurrent.locks;
import java.util.concurrent.TimeUnit;
import java.util.Collection;
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
    private final Sync sync;
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;
        abstract void lock();
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {//通过CAS去获取锁
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;//重入锁的实现
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;//兼容重入锁
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {//c=0才释放锁
                free = true;
                setExclusiveOwnerThread(null);//清除当前占用的线程
            }
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }
    }
    static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;
        final void lock() {
            if (compareAndSetState(0, 1))//通过CAS获取锁
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() && //判断队列中是否有前驱节点，如果有前驱节点，则获取不到锁
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

    /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();//默认非公平锁
    }
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

    public void lock() {
        sync.lock();
    }

    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
    //释放锁
    public void unlock() {
        sync.release(1);
    }
}
```

AbstractQueuedSynchronizer.java
```
package java.util.concurrent.locks;
import java.util.concurrent.TimeUnit;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import sun.misc.Unsafe;
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    private static final long serialVersionUID = 7373984972572414691L;

    protected AbstractQueuedSynchronizer() { }


    static final class Node {
        static final int PROPAGATE = -3;
        volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
        volatile Thread thread;
        Node nextWaiter;
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
    private transient volatile Node head;
    private transient volatile Node tail;
    private volatile int state;
    protected final int getState() {
        return state;
    }
    protected final void setState(int newState) {
        state = newState;
    }
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

    // Queuing utilities
    static final long spinForTimeoutThreshold = 1000L;
    //通过自旋的方式入队
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;//将当前线程排到末尾
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
    //将线程添加到等待队列
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;//将当前的队尾节点设置为自己的前驱
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);//插入队列，如果队列为空或者CAS失败
        return node;
    }
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }

    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {//如果是头节点的next则尝试获得锁
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())//避免无限的轮询
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
   private void unparkSuccessor(Node node) {
        int ws = node.waitStatus;//如果head当前的waitStatus小于0，将其修改为0
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;//从队尾往前，找到第一个waitStatus小于0的所有节点中排在最前面的
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)//唤醒线程
            LockSupport.unpark(s.thread);
    }
    protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
    public final void acquire(int arg) {//非公平锁回调用自己的tryAcquire方法，
        if (!tryAcquire(arg) &&         //如果没有获取到锁
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))//防止自旋
            selfInterrupt();
    }

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    //通过设置waitStatus避免无限轮询
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;//保存了前驱节点的状态，当线程需要挂起时，可以返回true
        if (ws == Node.SIGNAL)//为-1时，说明前驱节点状态正常
            return true;
        if (ws > 0) {//大于0表示前驱节点取消了排队
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    
    
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long stateOffset;
    private static final long headOffset;
    private static final long tailOffset;
    private static final long waitStatusOffset;
    private static final long nextOffset;

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

        } catch (Exception ex) { throw new Error(ex); }
    }
    private final boolean compareAndSetHead(Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, null, update);
    }
    private final boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }
    private static final boolean compareAndSetWaitStatus(Node node,
                                                         int expect,
                                                         int update) {
        return unsafe.compareAndSwapInt(node, waitStatusOffset,
                                        expect, update);
    }
    private static final boolean compareAndSetNext(Node node,
                                                   Node expect,
                                                   Node update) {
        return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
    }

    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                // 如果这个方法失败，会将节点设置为"取消"状态，并抛出异常 IllegalMonitorStateException
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
    //判断node是否已经移动到阻塞队列了
    final boolean isOnSyncQueue(Node node) {
        // 移动过去的时候，node 的 waitStatus 会置为 0
        // 如果 waitStatus 还是 Node.CONDITION，也就是 -2，那肯定就是还在条件队列中
        // 如果 node 的前驱 prev 指向还是 null，说明肯定没有在 阻塞队列(prev是阻塞队列链表中使用的)
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        // 如果 node 已经有后继节点 next 的时候，那肯定是在阻塞队列了
        if (node.next != null) // If has successor, it must be on queue
            return true;
        // 下面这个方法从阻塞队列的队尾开始从后往前遍历找，如果找到相等的，说明在阻塞队列，否则就是不在阻塞队列
        return findNodeFromTail(node);
    }
    // 从阻塞队列的队尾往前遍历，如果找到，返回 true
    private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }
    public class ConditionObject implements Condition, java.io.Serializable {
        private transient Node firstWaiter;
        private transient Node lastWaiter;
        //首先，这个方法是可被中断的，不可被中断的是另一个方法 awaitUninterruptibly()
        //这个方法会阻塞，直到调用 signal 方法（指 signal() 和 signalAll()，下同），或被中断
        public final void await() throws InterruptedException {
            if (Thread.interrupted())//判断中断状态
                throw new InterruptedException();
            Node node = addConditionWaiter();//添加到Condition的条件队列
            int savedState = fullyRelease(node);//释放锁，因为await之前，当前线程肯定是持有锁的
            int interruptMode = 0;//1-需要设置中断状态，0-没有发生中断，-1需要抛出异常
            while (!isOnSyncQueue(node)) {//返回True时，说明node已经转移到阻塞队列了
                LockSupport.park(this);//不在阻塞队列中，挂起线程
                //常规路径。signal -> 转移节点到阻塞队列 -> 获取了锁（unpark）
                //线程中断。在 park 的时候，另外一个线程对这个线程进行了中断
                //转移以后的前驱节点取消了，或者对前驱节点的CAS操作失败了
                //假唤醒
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)//代表当前线程中断
                    break;
            }
            //被唤醒以后，进入阻塞队列，等待获取锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
        // 将当前线程对应的节点入队，插入队尾
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {// 如果条件队列的最后一个节点取消了，将其清除出去
                unlinkCancelledWaiters();// 这个方法会遍历整个条件队列，然后会将已取消的所有节点清除出队列
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);// node 在初始化的时候，指定 waitStatus 为 Node.CONDITION
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
        // 等待队列是一个单向链表，遍历链表将已经取消等待的节点清除出去
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {//清除条件
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }
        public final void signal() {
            if (!isHeldExclusively())// 调用 signal 方法的线程必须持有当前的独占锁
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
        private void doSignal(Node first) {
            do {
                // 将 firstWaiter 指向 first 节点后面的第一个，因为 first 节点马上要离开了
                // 如果将 first 移除后，后面没有节点在等待了，那么需要将 lastWaiter 置为 null
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;// 因为 first 马上要被移到阻塞队列了，和条件队列的链接关系在这里断掉
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
            // 这里 while 循环，如果 first 转移不成功，那么选择 first 后面的第一个节点进行转移，依此类推
        }
        // 1. 如果在 signal 之前已经中断，返回 THROW_IE
        // 2. 如果是 signal 之后中断，返回 REINTERRUPT
        // 3. 没有发生中断，返回 0
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
    }
    // 将节点从条件队列转移到阻塞队列
    // true 代表成功转移
    // false 代表在 signal 之前，节点已经取消了
    final boolean transferForSignal(Node node) {
        // CAS 如果失败，说明此 node 的 waitStatus 已不是 Node.CONDITION，说明节点已经取消，
        // 既然已经取消，也就不需要转移了，方法返回，转移后面一个节点
        // 否则，将 waitStatus 置为 0
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;
        // enq(node): 自旋进入阻塞队列的队尾
        // 注意，这里的返回值 p 是 node 在阻塞队列的前驱节点
        Node p = enq(node);
        int ws = p.waitStatus;
        // ws > 0 说明 node 在阻塞队列中的前驱节点取消了等待锁，直接唤醒 node 对应的线程
        // 如果 ws <= 0, 那么 compareAndSetWaitStatus 将会被调用
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
    // 只有线程处于中断状态，才会调用此方法
    // 如果需要的话，将这个已经取消等待的节点转移到阻塞队列
    // 返回 true：如果此线程在 signal 之前被取消，
    final boolean transferAfterCancelledWait(Node node) {
        // 用 CAS 将节点状态设置为 0 
        // 如果这步 CAS 成功，说明是 signal 方法之前发生的中断，因为如果 signal 先发生的话，signal 中会将 waitStatus 设置为 0
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);
            // 将节点放入阻塞队列
            // 这里我们看到，即使中断了，依然会转移到阻塞队列
            return true;
        }
        // 到这里是因为 CAS 失败，肯定是因为 signal 方法已经将 waitStatus 设置为了 0
        // signal 方法会将节点转移到阻塞队列，但是可能还没完成，这边自旋等待其完成
        // 当然，这种事情还是比较少的吧：signal 调用之后，没完成转移之前，发生了中断
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }

}

```