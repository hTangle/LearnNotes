

ExecutorService.java
```
public interface ExecutorService extends Executor{
    // 关闭线程池，已提交的任务继续执行，不接受继续提交新任务
    void shutdown();
    // 关闭线程池，尝试停止正在执行的所有任务，不接受继续提交新任务
    // 它和前面的方法相比，加了一个单词“now”，区别在于它会去停止当前正在进行的任务
    List<Runnable> shutdownNow();
    boolean isShutdown();// 线程池是否已关闭
    // 如果调用了 shutdown() 或 shutdownNow() 方法后，所有任务结束了，那么返回true
    // 这个方法必须在调用shutdown或shutdownNow方法之后调用才会返回true
    boolean isTerminated();
    // 等待所有任务完成，并设置超时时间
    // 我们这么理解，实际应用中是，先调用 shutdown 或 shutdownNow，
    // 然后再调这个方法等待所有的线程真正地完成，返回值意味着有没有超时
    boolean awaitTermination(long timeout, TimeUnit unit)throws InterruptedException;
    // 提交一个 Callable 任务
    <T> Future<T> submit(Callable<T> task);
    // 提交一个 Runnable 任务，第二个参数将会放到 Future 中，作为返回值，
    // 因为 Runnable 的 run 方法本身并不返回任何东西
    <T> Future<T> submit(Runnable task, T result);
    Future<?> submit(Runnable task);// 提交一个 Runnable 任务
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;    
}
```

AbstractExecutorService.java
```
public abstract class AbstractExecutorService implements ExecutorService {
    // RunnableFuture 是用于获取执行结果的，我们常用它的子类 FutureTask
    // 下面两个 newTaskFor 方法用于将我们的任务包装成 FutureTask 提交到线程池中执行
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }  
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
    // 提交任务
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        // 1. 将任务包装成 FutureTask
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        // 2. 交给执行器执行，execute 方法由具体的子类来实现
        // 前面也说了，FutureTask 间接实现了Runnable 接口。
        execute(ftask);
        return ftask;
    }
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        // 1. 将任务包装成 FutureTask
        RunnableFuture<T> ftask = newTaskFor(task, result);
        // 2. 交给执行器执行
        execute(ftask);
        return ftask;
    }
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        // 1. 将任务包装成 FutureTask
        RunnableFuture<T> ftask = newTaskFor(task);
        // 2. 交给执行器执行
        execute(ftask);
        return ftask;
    }
    // 此方法目的：将 tasks 集合中的任务提交到线程池执行，任意一个线程执行完后就可以结束了
    // 第二个参数 timed 代表是否设置超时机制，超时时间为第三个参数，
    // 如果 timed 为 true，同时超时了还没有一个线程返回结果，那么抛出 TimeoutException 异常
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                              boolean timed, long nanos)
        throws InterruptedException, ExecutionException, TimeoutException {
        if (tasks == null)
            throw new NullPointerException();
        int ntasks = tasks.size();//任务个数
        if (ntasks == 0)
            throw new IllegalArgumentException();
        // ExecutorCompletionService 不是一个真正的执行器，参数 this 才是真正的执行器
        // 它对执行器进行了包装，每个任务结束后，将结果保存到内部的一个 completionQueue 队列中
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks);
        ExecutorCompletionService<T> ecs =
            new ExecutorCompletionService<T>(this);

        try {
            // 用于保存异常信息，此方法如果没有得到任何有效的结果，那么我们可以抛出最后得到的一个异常
            ExecutionException ee = null;
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Iterator<? extends Callable<T>> it = tasks.iterator();

            // Start one task for sure; the rest incrementally
             // 首先先提交一个任务，后面的任务到下面的 for 循环一个个提交
            futures.add(ecs.submit(it.next()));
            --ntasks;// 提交了一个任务，所以任务数量减 1
            int active = 1;// 正在执行的任务数(提交的时候 +1，任务结束的时候 -1)

            for (;;) {
                Future<T> f = ecs.poll();
                if (f == null) {// 为 null，说明刚刚提交的第一个线程还没有执行完成
                    if (ntasks > 0) {
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    else if (active == 0)//  这里的 active == 0，说明所有的任务都执行失败，那么这里是 for 循环出口
                        break;
                    else if (timed) {
                        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                        if (f == null)
                            throw new TimeoutException();
                        nanos = deadline - System.nanoTime();
                    }
                    // 这里是 else。说明，没有任务需要提交，但是池中的任务没有完成，还没有超时(如果设置了超时)
                    // take() 方法会阻塞，直到有元素返回，说明有任务结束了
                    else
                        f = ecs.take();
                }
                if (f != null) {// 有任务结束了
                    --active;
                    try {
                        return f.get();
                    } catch (ExecutionException eex) {
                        ee = eex;
                    } catch (RuntimeException rex) {
                        ee = new ExecutionException(rex);
                    }
                }
            }

            if (ee == null)
                ee = new ExecutionException();
            throw ee;

        } finally {
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
        }
    }
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
        try {
            return doInvokeAny(tasks, false, 0);
        } catch (TimeoutException cannotHappen) {
            assert false;
            return null;
        }
    }
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                           long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
        return doInvokeAny(tasks, true, unit.toNanos(timeout));
    }
    // 执行所有的任务，返回任务结果。
    // 先不要看这个方法，我们先想想，其实我们自己提交任务到线程池，也是想要线程池执行所有的任务
    // 只不过，我们是每次 submit 一个任务，这里以一个集合作为参数提交
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        List<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            // 这个很简单
            for (Callable<T> t : tasks) {
                // 包装成 FutureTask
                RunnableFuture<T> f = newTaskFor(t);
                futures.add(f);
                // 提交任务
                execute(f);
            }
            for (Future<T> f : futures) {
                if (!f.isDone()) {
                    try {
                        // 这是一个阻塞方法，直到获取到值，或抛出了异常
                        // 这里有个小细节，其实 get 方法签名上是会抛出 InterruptedException 的
                        // 可是这里没有进行处理，而是抛给外层去了。此异常发生于还没执行完的任务被取消了
                        f.get();
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    }
                }
            }
            done = true;
            // 这个方法返回，不像其他的场景，返回 List<Future>，其实执行结果还没出来
            // 这个方法返回是真正的返回，任务都结束了
            return futures;
        } finally {
            // 为什么要这个？就是上面说的有异常的情况
            if (!done)
                for (Future<T> f : futures)
                    f.cancel(true);
        }
    }
    // 带超时的 invokeAll，我们找不同吧
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
        if (tasks == null || unit == null)
            throw new NullPointerException();
        long nanos = unit.toNanos(timeout);
        List<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
        boolean done = false;
        try {
            for (Callable<T> t : tasks)
                futures.add(newTaskFor(t));

            long lastTime = System.nanoTime();

            Iterator<Future<T>> it = futures.iterator();
            // 提交一个任务，检测一次是否超时
            while (it.hasNext()) {
                execute((Runnable)(it.next()));
                long now = System.nanoTime();
                nanos -= now - lastTime;
                lastTime = now;
                // 超时
                if (nanos <= 0)
                    return futures;
            }

            for (Future<T> f : futures) {
                if (!f.isDone()) {
                    if (nanos <= 0)
                        return futures;
                    try {
                        // 调用带超时的 get 方法，这里的参数 nanos 是剩余的时间，
                        // 因为上面其实已经用掉了一些时间了
                        f.get(nanos, TimeUnit.NANOSECONDS);
                    } catch (CancellationException ignore) {
                    } catch (ExecutionException ignore) {
                    } catch (TimeoutException toe) {
                        return futures;
                    }
                    long now = System.nanoTime();
                    nanos -= now - lastTime;
                    lastTime = now;
                }
            }
            done = true;
            return futures;
        } finally {
            if (!done)
                for (Future<T> f : futures)
                    f.cancel(true);
        }
    }
}
```
ThreadPoolExecutor.java
```
public class ThreadPoolExecutor extends AbstractExecutorService {
    public ThreadPoolExecutor(int corePoolSize,//核心线程数
                              int maximumPoolSize,//最大线程数
                              long keepAliveTime,//空闲存活时间
                              TimeUnit unit,//
                              BlockingQueue<Runnable> workQueue,//任务队列
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
    // 这里 COUNT_BITS 设置为 29(32-3)，意味着前三位用于存放线程状态，后29位用于存放线程数
    // 很多初学者很喜欢在自己的代码中写很多 29 这种数字，或者某个特殊的字符串，然后分布在各个地方，这是非常糟糕的
    private static final int COUNT_BITS = Integer.SIZE - 3;
    // 000 11111111111111111111111111111
    // 这里得到的是 29 个 1，也就是说线程池的最大线程数是 2^29-1=536870911
    // 以我们现在计算机的实际情况，这个数量还是够用的
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    // 我们说了，线程池的状态存放在高 3 位中
    // 运算结果为 111跟29个0：111 00000000000000000000000000000
    private static final int RUNNING    = -1 << COUNT_BITS;
    // 000 00000000000000000000000000000
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 001 00000000000000000000000000000
    private static final int STOP       =  1 << COUNT_BITS;
    // 010 00000000000000000000000000000
    private static final int TIDYING    =  2 << COUNT_BITS;
    // 011 00000000000000000000000000000
    private static final int TERMINATED =  3 << COUNT_BITS;
    // 将整数 c 的低 29 位修改为 0，就得到了线程池的状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    // 将整数 c 的高 3 为修改为 0，就得到了线程池中的线程数
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        private static final long serialVersionUID = 6138294804551838833L;
        final Thread thread;
        Runnable firstTask;
        volatile long completedTasks;// 用于存放此线程完全的任务数，注意了，这里用了 volatile，保证可见性
        // Worker 只有这一个构造方法，传入 firstTask，也可以传 null
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);// 调用 ThreadFactory 来创建一个新的线程
        }
        public void run() {
            runWorker(this);
        }
    }
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            // 如果当前线程数少于核心线程数，那么直接添加一个 worker 来执行任务，
            // 创建一个新的线程，并把当前任务 command 作为这个线程的第一个任务(firstTask)
            // 添加任务成功，那么就结束了。提交任务嘛，线程池已经接受了这个任务，这个方法也就可以返回了
            // 至于执行的结果，到时候会包装到 FutureTask 中。
            // 返回 false 代表线程池不允许提交任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 如果线程池处于 RUNNING 状态，把这个任务添加到任务队列 workQueue 中
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 如果线程池已不处于 RUNNING 状态，那么移除已经入队的这个任务，并且执行拒绝策略
            if (! isRunning(recheck) && remove(command))
                reject(command);
            // 如果线程池还是 RUNNING 的，并且线程数为 0，那么开启新的线程
            // 到这里，我们知道了，这块代码的真正意图是：担心任务提交到队列中了，但是线程都关闭了
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 如果 workQueue 队列满了，那么进入到这个分支
        // 以 maximumPoolSize 为界创建新的 worker，
        // 如果失败，说明当前线程数已经达到 maximumPoolSize，执行拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
    // 第一个参数是准备提交给这个线程执行的任务，之前说了，可以为 null
    // 第二个参数为 true 代表使用核心线程数 corePoolSize 作为创建线程的界线，也就说创建这个线程的时候，
    // 如果线程池中的线程总数已经达到 corePoolSize，那么不能响应这次创建线程的请求
    // 如果是 false，代表使用最大线程数 maximumPoolSize 作为界线
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&//如果线程池状态大于shutdown
                ! (rs == SHUTDOWN &&//
                   firstTask == null &&
                   ! workQueue.isEmpty()))// 当状态大于 SHUTDOWN 时，不允许提交任务，且中断正在执行的任务
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 如果成功，那么就是所有创建线程前的条件校验都满足了，准备创建线程执行任务了
                // 这里失败的话，说明有其他线程也在尝试往线程池中创建线程
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 由于有并发，重新再读取一下 ctl
                c = ctl.get();  // Re-read ctl
                // 正常如果是 CAS 失败的话，进到下一个里层的for循环就可以了
                // 可是如果是因为其他线程的操作，导致线程池的状态发生了变更，如有其他线程关闭了这个线程池
                // 那么需要回到外层的for循环
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;// worker 是否已经启动
        boolean workerAdded = false;// 是否已将这个 worker 添加到 workers 这个 HashSet 中
        Worker w = null;
        try {
            w = new Worker(firstTask);// 把 firstTask 传给 worker 的构造方法
            final Thread t = w.thread;// 取 worker 中的线程对象，之前说了，Worker的构造方法会调用 ThreadFactory 来创建一个新的线程
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    int rs = runStateOf(ctl.get());
                    // 小于 SHUTTDOWN 那就是 RUNNING，这个自不必说，是最正常的情况
                    // 如果等于 SHUTDOWN，前面说了，不接受新的任务，但是会继续执行等待队列中的任务
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();// 添加成功的话，启动这个线程
                    workerStarted = true;
                }
            }
        } finally {
            // 如果线程没有启动，需要做一些清理工作，如前面 workCount 加了 1，将其减掉
            if (! workerStarted)
                addWorkerFailed(w);
        }
        // 返回线程是否启动成功
        return workerStarted;
    }
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            // 循环调用 getTask 获取任务
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 如果线程池状态大于等于 STOP，那么意味着该线程也要中断
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;// 置空 task，准备 getTask 获取下一个任务
                    w.completedTasks++;// 累加完成的任务数
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
    // 此方法有三种可能：
    // 1. 阻塞直到获取到任务返回。我们知道，默认 corePoolSize 之内的线程是不会被回收的，
    //      它们会一直等待任务
    // 2. 超时退出。keepAliveTime 起作用的时候，也就是如果这么多时间内都没有任务，那么应该执行关闭
    // 3. 如果发生了以下条件，此方法必须返回 null:
    //    - 池中有大于 maximumPoolSize 个 workers 存在(通过调用 setMaximumPoolSize 进行设置)
    //    - 线程池处于 SHUTDOWN，而且 workQueue 是空的，前面说了，这种不再接受新的任务
    //    - 线程池处于 STOP，不仅不接受新的线程，连 workQueue 中的线程也不再执行
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 两种可能
            // 1. rs == SHUTDOWN && workQueue.isEmpty()
            // 2. rs >= STOP
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();// CAS 操作，减少工作线程数
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }
            // wc <= maximumPoolSize 同时没有超时
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
    // 只要线程池没有被关闭，那么由提交任务的线程自己来执行这个任务。
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        public CallerRunsPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

    // 不管怎样，直接抛出 RejectedExecutionException 异常
    // 这个是默认的策略，如果我们构造线程池的时候不传相应的 handler 的话，那就会指定使用这个
    public static class AbortPolicy implements RejectedExecutionHandler {
        public AbortPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                " rejected from " +
                                                e.toString());
        }
    }

    // 不做任何处理，直接忽略掉这个任务
    public static class DiscardPolicy implements RejectedExecutionHandler {
        public DiscardPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }

    // 这个相对霸道一点，如果线程池没有被关闭的话，
    // 把队列队头的任务(也就是等待了最长时间的)直接扔掉，然后提交这个任务到等待队列中
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        public DiscardOldestPolicy() { }
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
}
```