## 线程池
**Executor->ExecutorService->AbstractExecutorService->ThreadPoolExecutor**
> Callable 也是因为线程池的需要，所以才有了这个接口。它和 Runnable 的区别在于 run() 没有返回值，而 Callable 的 call() 方法有返回值，同时，如果运行出现异常，call() 方法会抛出异常。

**Future   -> RunnableFuture -> FutureTask**
**Runnable -> RunnableFuture**
**FutureTask 通过 RunnableFuture 间接实现了 Runnable 接口**

#### ThreadPoolExecutor类的必要参数
1. corePoolSize：核心线程数
2. maximumPoolSize：最大线程数
3. workQueue：任务队列
4. keepAliveTime：空闲线程存活时间，线程数大于核心线程数时起作用，也可以设置allowCoreThreadTimeOut(true)对核心线程起作用
5. threadFactory：生成线程的线程工厂
6. handle：线程池满时的执行策略
   1. CallerRunsPolicy:只要线程池没有被关闭，那么由提交任务的线程自己来执行这个任务
   2. AbortPolicy:不管怎样，直接抛出 RejectedExecutionException 异常(默认)
   3. DiscardPolicy：不做任何处理，直接忽略掉这个任务
   4. DiscardOldestPolicy：如果线程池没有被关闭的话，把队列队头的任务(也就是等待了最长时间的)直接扔掉，然后提交这个任务到等待队列中

#### 线程池的状态 大于0不能提交任务
1. RUNNING(-1)：接受新的任务，处理等待队列中的任务
2. SHUTDOWN：不接受新的任务提交，但是会继续处理等待队列中的任务
3. STOP：不接受新的任务提交，不再处理等待队列中的任务，中断正在执行任务的线程(shutDownNow())
4. TIDYING：所有的任务都销毁了，workCount 为 0。线程池的状态在转换为 TIDYING 状态时，会执行钩子方法 terminated()
5. TERMINATED：terminated() 方法结束后，线程池的状态就会变成这个

### rejectedExecutionHandler 用于处理当线程池不能执行此任务时的情况，默认有抛出 RejectedExecutionException 异常、忽略任务、使用提交任务的线程来执行此任务和将队列中等待最久的任务删除，然后提交此任务这四种策略，默认为抛出异常。

#### 线程池中的线程创建时机
1. 如果当前线程数少于 corePoolSize，那么提交任务的时候创建一个新的线程，并由这个线程执行这个任务；
2. 如果当前线程数已经达到 corePoolSize，那么将提交的任务添加到队列中，等待线程池中的线程去队列中取任务；
3. 如果队列已满，那么创建新的线程来执行任务，需要保证池中的线程数不会超过 maximumPoolSize，如果此时线程数超过了 maximumPoolSize，那么执行拒绝策略。
**注意：如果将队列设置为无界队列，那么线程数达到 corePoolSize 后，其实线程数就不会再增长了。**
* 第1种是：固定大小线程池，特点是线程数固定，使用无界队列，适用于任务数量不均匀的场景、对内存压力不敏感，但系统负载比较敏感的场景；
* 第2种是：Cached线程池，特点是不限制线程数，适用于要求低延迟的短期任务场景；
* 第3种是：单线程线程池，也就是一个线程的固定线程池，适用于需要异步执行但需要保证任务顺序的场景；
* 第4种是：Scheduled线程池，适用于定期执行任务场景，支持按固定频率定期执行和按固定延时定期执行两种方式；
* 第5种是：工作窃取线程池，使用的ForkJoinPool，是固定并行度的多任务队列，适合任务执行时长不均匀的场景。
#### newFixedThreadPool 最大线程数设置为与核心线程数相等，此时 keepAliveTime 设置为 0（因为这里它是没用的，即使不为 0，线程池默认也不会回收 corePoolSize 内的线程），任务队列采用 LinkedBlockingQueue，无界队列。刚开始，每提交一个任务都创建一个 worker，当 worker 的数量达到 nThreads 后，不再创建新的线程，而是把任务提交到 LinkedBlockingQueue 中，而且之后线程数始终为 nThreads(适用于任务数量不均，对内存压力不敏感但对系统负载敏感的场景)
#### newCachedThreadPool:核心线程数为 0，最大线程数为 Integer.MAX_VALUE，keepAliveTime 为 60 秒，任务队列采用 SynchronousQueue。这种线程池对于任务可以比较快速地完成的情况有比较好的性能。如果线程空闲了 60 秒都没有任务，那么将关闭此线程并从线程池中移除。所以如果线程池空闲了很长时间也不会有问题，因为随着所有的线程都会被关闭，整个线程池不会占用任何的系统资源。
#### 任务执行过程中发生异常:如果某个任务执行出现异常，那么执行任务的线程会被关闭，而不是继续接收其他任务。然后会启动一个新的线程来代替它。
#### 执行拒绝策略 
1. workers 的数量达到了 corePoolSize（任务此时需要进入任务队列），任务入队成功，与此同时线程池被关闭了，而且关闭线程池并没有将这个任务出队，那么执行拒绝策略。这里说的是非常边界的问题，入队和关闭线程池并发执行，读者仔细看看 execute 方法是怎么进到第一个 reject(command) 里面的。
2. workers 的数量大于等于 corePoolSize，将任务加入到任务队列，可是队列满了，任务入队失败，那么准备开启新的线程，可是线程数已经达到 maximumPoolSize，那么执行拒绝策略。

### 进程和线程
#### 进程：就绪，运行，阻塞
1. **匿名管道**：半双工，父子或兄弟进程之间，速度慢，容量受限
2. **命名管道**：无关进程之间的通信，有路径关联，速度慢
3. **消息队列**：消息的链接表，存放于内核，使用标识符标识，消息队列独立于发送与接收进程，进程终止时，消息队列及其内容并不会删除，消息队列可以实现消息的随机查询，消息不一定要以先进先出的次序读取，可以按照消息类型读取
4. **信号量**：是一个计数器，用于同步或互斥
5. **共享内存**：速度快，进程是直接对内存进行读取的，需要同步
* 守护进程：控制终端并周期性执行任务或处理事件，不受用户登录或注销的影响
* 孤儿进程：父程序先退出了，子程序托付给了Init进程
* 僵尸进程：进程终止，但是父程序未获取其状态，需要保留部分信息

### 线程之间的通信方式
* 同步：synchronized
* while轮询
* wait notify
* 管道
* 共享内存