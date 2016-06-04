# ThreadPoolExecutor解析

[TOC]

## ThreadPoolExecutor成员变量
- **AtomicInteger ctl：**用于记录线程池的状态和worker数，高三位为状态，后面为worker数
- **corePoolSize：**核心线程数，即使线程空闲也会保持，除非allowCoreThreadTimeOut为true的时候会回收
- **maximumPoolSize：**最大允许的线程数
- **keepAliveTime：**如果当前线程数大于核心线程数，这个值就是取任务的超时时间，如果超时，当前线程结束
- **unit：**时间单位
- **workQueue：** 用于存放任务的队列，只存放通过submit方法传入的任务
- **threadFactory：**线程工厂，用于自定义生成线程组或者优先线程等
- **RejectedExecutionHandler：**指定当workQueue满后的任务拒绝策略，默认有四种拒绝策略
 * AbortPolicy：直接抛RejectedExecutionException
 * DiscardPolicy：拒绝当前任务
 * DiscardOldestPolicy：拒绝队列里最老的任务
 * CallerRunsPolicy：调用execute的线程直接执行任务

## ThreadPoolExecutor.Worker内部类解析
* 继承了AbstractQueuedSynchronizer，一种锁的实现，从而使Worker本身变为一个锁
* 成员变量thread持有当前线程的引用，当前线程是通过threadFactory生成
* threadFactory可以用户自定义也可以默认，用户区分线程组，线程优先级等
* Worker的构造函数设置第一次任务：firstTask
* firstTask是指线程池新建新的线程的时候用于第一次线程启动时执行，第一次并不是从queue里面取任务，这点请谨记
* worker.run()->runWorker
 * 判断当前的threadPool是不是stop以前当前线程是不是被中断了，如果是则触发worker线程中断
 * 先执行beforeExecute再执行task的run方法，最后执行afterExecute，before和after方法用户可自己定义
 * 最终会执行processWorkerExit
 * 判断completedAbruptly是不是true(任务突然完成，其实是没有走task.run(),workQueue里面没有任务)
  * 如果是true,

## ThreadPoolExecutor.getTask()内部实现
* 如果当前线程池的状态是关闭后的状态并且workQueue为空的时候，workers的数量减一，`最终在runWorker的finally的方法里面把worker从workSet里面移除`
* timed:核心线程是否有超时限制或者worker数已大于核心线程数
> **if (wc <= maximumPoolSize && ! (timedOut && timed))表示worker数小于最大线程数，并且（上次循环取任务超时或者timed）就跳出子循环，到下在的取任务环节。**

* 如下符合上面的条件,work数量减1，即ctl减1 
* workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)取出的值为空的时候，再次循环的时候会到这个语句（if (wc <= maximumPoolSize && ! (timedOut && timed))），这个语句会不成立，然后接着走下一句compareAndDecrementWorkerCount，再返回null，getTask方法结束;

## ThreadPoolExecutor.execute(Runnable)执行顺序解析
1.   先取ctl的值，判断是否小于corePoolSize
2.  如果小于corePoolSize
* 调用addWorker
> addWorker的时候是不会从queue里面取任务执行的

 * 判断中断标志，shutDownNow可能随时触发中断，这里有个shutDownNow的竞争，重新设置中断标志，用户可以task.run()方法里针对这个中断标识判断是否需要停止当前任务
 * TODO
* 在addWorker方法里面判断队列及线程池的状态
* 接着ctl加一，创建一个新的worker
* 使用线程池内的成员变量mainLock，使得创建新的worker是一个同步过程
* 把worker加入到线程池的workers里面
* 校验workers的size是否超过largestPoolSize,如果大于，则largestPoolSize变为当前workers的数量
* 释放锁，再启动线程，启动的线程即为worker创建时由threadFactory生成的thread.
`如果启动失败，会把当前worker从workers里移除，ctl相应的CAS减1，尝试terminate线程池`

3. 校验线程池是否处于running状态，并且把command放入workQueue
 * 再次检查线程池的状态，如不处在running状态则从queue里移除command，再reject这个command 
 > reject会调用rejectedExecution用户自定义的处理


## ThreadPoolExecutor类说明
ExecutorService用来执行每一个提交的任务，这个机制是通过缓存着的线程实现，一般是通过Executors的工厂方法来配置ExecutorService

线程池可解决两种问题：减少了每个任务的调用开销，为大量异步任务提供更高的性能。并且提供了一系列手段去限制和管理各种资源和线程消耗。每个ThreadPoolExecutor会提供一些基础的统计信息，例如完成的任务数。

这个类提供了很多可调的参数和可伸缩的回调来确保在大规模的上下文中的可用性，然而我们更倾向于使用更方便的工厂方法：Executors.newCachedThreadPool（无边界，自动线程回收），Executor.newFixedThreadPool（固定长度的线程池），Executors.newSingleThreadExecutor（单线程后台线程）这些已经预定义好常用的配置，如果不想使用这几个工厂方法的话可以手动配置调优这个类：

**Core and maximum pool sizes(核心及最大线程数)**
线程池会通过corePoolSize和maximumPoolSize自动配置池的大小。
当一个新的任务通过execute提交时，如果当前处于running的线程数少于corePoolSize，即使有线程处于空闲状态依然会创建一个新的线程去执行任务。如果当前running的线程数已大于corePoolSize，但是少于maximumPoolSize，新的线程只有在`队列满`的时候创建。通过配置corePoolSize和maximumPoolSize两个参数，你会得到一个固定线程数的线程池。通过设置maximumPoolSize本质上允许Integer.MAX_VALUE的线程数（这种说明不太准确[^maxthreadcount]），通常core和maximum的线程数是通过构造函数设置，也可以通过set方法动态改变这两个值。

**On-demand construction(按需构建) ** 
核心线程默认是通过submit新的任务的时候创建，但是也可以通过prestartCoreThread或者prestartAllCoreThreads预启动。当初始队列不为空的时候，预启动核心线程将很管用。

**创建新线程**
新线程是通过ThreadFactory来创建。如不指定则使用Executors.defaultThreadFactory来创建，这样所有的线程都将是同一个线程组、同一种优先级以及非后台状态。通过定制的线程池，你可以定义线程的名称、线程组、优先级、前后台状态等。当ThreadFactory创建线程失败时，一般是返回null，这样可能会无法执行任何任务。这里面的线程应该需用拥有可更改线程的权限。如果worker线程或其他使用线程池的线程没有这个权限，服务应该降级：及时更改配置不一定会立刻生效，关闭线程池也可能会在终结状态，但也可能没有完成。

**Keep-alive times(存活时间)**
如果当前线程池有超过corePoolSize的活跃线程，而且活动线程空闲时间超过了存活时间，这个线程将会终结。这样可以减少资源消耗，如果线程池的任务提交变得更加活跃的时候，新的线程将会创建。这个参数也可以通过setKeepAliveTime动态改变，也可以设置Long的最大值使得即使空闲的线程也会一直保持。一般来说keep-alive的策略是用在线程数大于corePoolSize的时候起效，但是通过这个参数allowCoreThreadTimeOut也能作用于core threads（只要keepAliveTime不是0）。

**队列**
任何一个BlockingQueue都可以传输和存储提交的任务，队列的用法和线程池的大小息息相关。
如果小于corePoolSize的线程处于running的状态，Executor会倾向于增加新的线程而不是放入队列。
如果处于running的线程数大于等于corePoolSize，Executor会倾向于放入队列，而不是增加新的线程（队列满后则需要创建新的线程）。
如果请求无法放入队列，新的线程将会创建，直到到达maximumPoolSize最大的线程数，达到最大后，所接收的请求将会被拒绝。
下面是三个通用的队列策略：
`直接交付(Direct handoffs)`，默认好的选择是SynchronousQueue，这个队列直接交付任务，不会存储任务。如果在没有可用线程的情况下将任务放入队列，这将会失败，新的线程需要被创建。这个策略在处理一些内部有相互的依赖的请求的时候避免了锁的情况。直接交付通常需要一个无边界的最大线程数（maximumPoolSizes）以防止新的任务被拒绝。相反，在请求持续高于平均处理能力的时候会导致无法预估的线程增长。
`无界队列(unbounded queues)`，默认选择是LinkedBlockingQueue（没有定义容量），这种列队在核心线程都处理忙碌状态的时候可以使当前请求的任务排队等待，并不会创建超过核心线程之外的线程（maximumPoolSize这个参数将不会起作用）。这种队列适合任务间相互独立的场景，例如一个网页服务，当请求量爆发时，超过了平均处理能力，就先把任务入列。
`有界队列(bounded queues)`例如ArrayBlockingQueue，当线程的maximumPoolSize有限制的时候可以有效避免资源耗尽的情况发生，但是也很难做好线程池的调优和控制。队列的长度和线程池的长度会有相互妥协的情况发生：设定队列长度非常大，线程池的大小很小，可以防止过度的cpu的使用率、系统资源和上下文切换，但却会导致吞吐量下降。如果任务经常性的堵塞（例如I/O等待）系统应该能够安排时间片给更多的线程而不仅仅是你指定的线程。使用小的队列通常需要一个大的线程池，这样可以使CPU一直保持忙碌状态，但可能会遇到不可忍受的调度开销，这样也会导致吞吐量下降。
`拒绝的任务(rejected tasks)`
当Executor关闭后，通过execute方法进来的新任务将会被拒绝，同样，当Executor使用有界队列和设定了最大线程数，当两者都达到最大时任务也会被拒绝。无论哪种情况，execute方法将会调用RejectedExecutionHandler.rejectedExecution，目前定义了四种拒绝策略：
默认为ThreadPoolExecutor.AbortPolicy，抛出一个运行时的异常（RejectedExecutionException）
ThreadPoolExecutor.CallerRunsPolicy，调用execute方法的线程直接执行这个任务，这样的反馈控制机制可以减缓任务提交的速率。
ThreadPoolExecutor.DiscardPolicy，任务被简单的丢弃。
ThreadPoolExecutor.DiscardOldestPolicy，丢弃队列最前面的任务，重新执行当前任务（其他任务的提交还可能导致当前任务失败再次重试）
也可以自己定义其他的拒绝策略，设计时需要非常谨慎的考虑到系统容量和队列策略。

**回调函数(hook)**
这个类提供了可复写的beforeExecute和afterExecute方法，在每个任务的前后会被调用。这样可以为每个任务生成一个执行环境，例如：重新初始化threadLocal，获取统计数据或者加一些日志，除此之外，方法terminated也可以被重写，为的是能够在线程池关闭的时候执行些特殊的处理。如果回调函数抛出异常，内部的worker线程可能会失败，或者突然终结。

**队列维护**
getQueue方法允许取得当前队列，用于监控和调试，不鼓励其他用途，当队列的任务非常大的时候可使用remove和purge方法取消任务。

**终结**
当线程池不再被引用到并且没有工作线程，线程池将会自己shutdown，如果你想确保没有引用的线程池被回收，你需要设定好这些线程最终会结束（通过设置keep alive time、设定核心线程数为0或者allowCoreThreadTimeOut）。



[^maxthreadcount]: 因为高三位是留给了线程池的状态位，正确的线程数为$ 2^{29}-1$