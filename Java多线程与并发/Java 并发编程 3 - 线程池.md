# Java 并发编程 3 - 线程池



## 七、共享模型之工具

### 1、线程池

池化技术相比大家已经屡见不鲜了，线程池、数据库连接池、Http 连接池等等都是对这个思想的应用。池化技术的思想主要是为了减少每次获取资源的消耗，提高对资源的利用率。
线程池提供了一种限制和管理资源（包括执行一个任务）。 每个线程池还维护一些基本统计信息，例如已完成任务的数量。
这里借用《Java [并发编程](https://so.csdn.net/so/search?q=并发编程&spm=1001.2101.3001.7020)的艺术》提到的来说一下使用线程池的好处：

- 降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
- 提高响应速度。当任务到达时，任务可以不需要的等到线程创建就能立即执行。
- 提高线程的可管理性。线程是稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配，调优和监控。

### 2、自定义线程池

![在这里插入图片描述](../笔记图片/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70.png)
上图就是一个线程池的实现，先初始化线程池、阻塞队列大小，然后开几个线程通过线程池对象调用方法执行任务，线程池中的线程会执行任务，如果任务过多，会添加到阻塞队列中，执行完任务再从阻塞队列中取值继续执行。当执行的线程数大于线程池和阻塞队列的大小，我们可以定义拒绝策略，类似 jdk 线程池那样。代码实现如下：

```java
/**
 * 自定义线程池
 */
@Slf4j(topic = "c.Code_01_TestPool")
public class Code_01_ThreadPoolTest {

    public static void main(String[] args) {
        ThreadPool threadPool = new ThreadPool(1, 1000, TimeUnit.MILLISECONDS, 1,
                (queue, task) -> {
                    // 1. 阻塞等待。
//                    queue.put(task);
                    // 2. 带超时的等待
//                    queue.offer(task, 500, TimeUnit.MILLISECONDS);
                    // 3. 调用者放弃
//                    log.info("放弃 {}", task);
                    // 4. 调用者抛出异常
//                    throw new RuntimeException("任务执行失败" + task);
                    // 5. 调用者自己执行任务
                    task.run();
                });
        for(int i = 0; i < 4; i++) {
            int j = i;
            threadPool.executor(() ->{
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.info("{}", j);
            });
        }
    }

}

@FunctionalInterface // 拒绝策略
interface RejectPolicy<T> {
    void reject(BlockingQueue<T> queue, T task);
}

// 实现线程池
@Slf4j(topic = "c.ThreadPool")
class ThreadPool {
    // 线程集合
    private Set<Worker> works = new HashSet<Worker>();
    // 任务队列
    private BlockingQueue<Runnable> taskQueue;
    // 线程池的核心数
    private int coreSize;
    // 获取任务的超时时间
    private long timeout;
    private TimeUnit unit;
    // 使用策略模式。
    private RejectPolicy<Runnable> rejectPolicy;

    public ThreadPool(int coreSize, long timeout, TimeUnit unit, int queueCapacity,
                      RejectPolicy<Runnable> rejectPolicy) {
        this.coreSize = coreSize;
        this.timeout = timeout;
        this.unit = unit;
        taskQueue = new BlockingQueue<>(queueCapacity);
        this.rejectPolicy = rejectPolicy;
    }

    // 执行任务
    public void executor(Runnable task) {
        // 如果线程池满了. 就将任务加入到任务队列, 否则执行任务
        synchronized (works) {
            if(works.size() < coreSize) {
                Worker worker = new Worker(task);
                log.info("新增 worker {} ，任务 {}", worker, task);
                works.add(worker);
                worker.start();
            } else {
//                taskQueue.put(task);
                // 1）死等
                // 2）带超时等待
                // 3）让调用者放弃任务执行
                // 4）让调用者抛出异常
                // 5）让调用者自己执行任务

                taskQueue.tryPut(rejectPolicy, task);
            }
        }
    }

    class Worker extends Thread {

        private Runnable task;

        public Worker(Runnable task) {
            this.task = task;
        }

        // 执行任务
        // 1）当 task 不为空，执行任务
        // 2）当 task 执行完毕，再接着从任务队列获取任务并执行
        @Override
        public void run() {
//            while (task != null || (task = taskQueue.take()) != null) {
            while (task != null || (task = taskQueue.poll(timeout, unit)) != null) {
                 try { 
                     log.info("正在执行 {}", task);
                     task.run();
                 }catch (Exception e) {

                 } finally {
                     task = null;
                 }
            }
            synchronized (works) {
                log.info("worker 被移除 {}", this);
                works.remove(this);
            }
        }
    }
}
// 实现阻塞队列
@Slf4j(topic = "c.BlockingQueue")
class BlockingQueue<T> {

    // 阻塞队列的容量
    private int capacity;
    // 双端链表, 从头取, 从尾加
    private Deque<T> queue;
    // 定义锁
    private ReentrantLock lock;
    // 当阻塞队列满了时候, 去 fullWaitSet 休息, 生产者条件变量
    private Condition fullWaitSet;
    // 当阻塞队列空了时候，去 emptyWaitSet 休息, 消费者小件变量
    private Condition emptyWaitSet;

    public BlockingQueue(int capacity) {
        queue = new ArrayDeque<>(capacity);
        lock = new ReentrantLock();
        fullWaitSet = lock.newCondition();
        emptyWaitSet = lock.newCondition();
        this.capacity = capacity;
    }

    // 带有超时时间的获取
    public T poll(long timeout, TimeUnit unit) {
        lock.lock();
        try {
            // 同一时间单位
            long nanos = unit.toNanos(timeout);
            while (queue.isEmpty()) {
                try {
                    if(nanos <= 0) {
                        return null;
                    }
                    // 防止虚假唤醒, 返回的是所剩时间
                    nanos = emptyWaitSet.awaitNanos(nanos);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            T t = queue.removeFirst();
            fullWaitSet.signal();
            return t;
        }finally {
            lock.unlock();
        }
    }

    // 获取
    public T take() {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                try {
                    emptyWaitSet.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            T t = queue.removeFirst();
            fullWaitSet.signal();
            return t;
        }finally {
            lock.unlock();
        }
    }

    // 添加
    public void put(T task) {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                try {
                    log.info("等待加入任务队列 {}", task);
                    fullWaitSet.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.info("加入任务队列 {}", task);
            queue.addLast(task);
            emptyWaitSet.signal();
        }finally {
            lock.unlock();
        }
    }
    // 带有超时时间的添加
    public boolean offer(T task, long timeout, TimeUnit unit) {
        lock.lock();
        try {
            long nanos = unit.toNanos(timeout);
            while (queue.size() == capacity) {
                try {
                    if(nanos <= 0) {
                        return false;
                    }
                    log.info("等待加入任务队列 {}", task);
                    nanos = fullWaitSet.awaitNanos(nanos);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.info("加入任务队列 {}", task);
            queue.addLast(task);
            emptyWaitSet.signal();
            return true;
        }finally {
            lock.unlock();
        }
    }

    public void tryPut(RejectPolicy<T> rejectPolicy, T task) {
        lock.lock();
        try {
            // 判断判断是否满
            if(queue.size() == capacity) {
                rejectPolicy.reject(this, task);
            } else { // 有空闲
                log.info("加入任务队列 {}", task);
                queue.addLast(task);
                emptyWaitSet.signal();
            }

        }finally {
            lock.unlock();
        }
    }

    public int getSize() {
        lock.lock();
        try {
            return queue.size();
        } finally {
            lock.unlock();
        }
    }
}

123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687888990919293949596979899100101102103104105106107108109110111112113114115116117118119120121122123124125126127128129130131132133134135136137138139140141142143144145146147148149150151152153154155156157158159160161162163164165166167168169170171172173174175176177178179180181182183184185186187188189190191192193194195196197198199200201202203204205206207208209210211212213214215216217218219220221222223224225226227228229230231232233234235236237238239240241242243244245246247248249250251252253254255
```

### 3、ThreadPoolExecutor

#### 1）线程池状态

ThreadPoolExecutor 使用 int 的高 3 位来表示线程池状态，低 29 位表示线程数量，ThreadPoolExecutor 类中的线程状态变量如下：

```java
	// Integer.SIZE 值为 32 
  	private static final int COUNT_BITS = Integer.SIZE - 3;
    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
12345678
```

| 状态名称   | 高3位的值 | 描述                                          |
| ---------- | --------- | --------------------------------------------- |
| RUNNING    | 111       | 接收新任务，同时处理任务队列中的任务          |
| SHUTDOWN   | 000       | 不接受新任务，但是处理任务队列中的任务        |
| STOP       | 001       | 中断正在执行的任务，同时抛弃阻塞队列中的任务  |
| TIDYING    | 010       | 任务执行完毕，活动线程为0时，即将进入终结阶段 |
| TERMINATED | 011       | 终结状态                                      |

线程池状态和线程池中线程的数量**由一个原子整型ctl来共同表示**

- 使用一个数来表示两个值的主要原因是：**可以通过一次CAS同时更改两个属性的值**

```java
// 原子整数，前 3 位保存了线程池的状态，剩余位保存的是线程数量
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

// 并不是所有平台的 int 都是 32 位。
// 去掉前三位保存线程状态的位数，剩下的用于保存线程数量
// 高3位为0，剩余位数全为1
private static final int COUNT_BITS = Integer.SIZE - 3;

// 2^COUNT_BITS次方，表示可以保存的最大线程数
// CAPACITY 的高3位为 0
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
1234567891011
```

获取线程池状态、线程数量以及合并两个值的操作

```java
// Packing and unpacking ctl
// 获取运行状态
// 该操作会让除高3位以外的数全部变为0
private static int runStateOf(int c)     { return c & ~CAPACITY; }

// 获取运行线程数
// 该操作会让高3位为0
private static int workerCountOf(int c)  { return c & CAPACITY; }

// 计算ctl新值
private static int ctlOf(int rs, int wc) { return rs | wc; }
1234567891011
```

**线程池的属性：**

```java
// 工作线程，内部封装了Thread
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {
    ...
}

// 阻塞队列，用于存放来不及被核心线程执行的任务
private final BlockingQueue<Runnable> workQueue;

// 锁
private final ReentrantLock mainLock = new ReentrantLock();

//  用于存放核心线程的容器，只有当持有锁时才能够获取其中的元素（核心线程）
private final HashSet<Worker> workers = new HashSet<Worker>();
123456789101112131415
```

#### 2）构造方法

首先我们看一下 ThreadPoolExecutor 类参数最多、最全的有参构造方法。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
1234567
```

**构造参数解释：**

- corePoolSize：核心线程数
- maximumPoolSize：最大线程数
  - maximumPoolSize - corePoolSize = 救急线程数
- keepAliveTime：救急线程空闲时的最大生存时间
- unit：时间单位
- workQueue：阻塞队列（存放任务）
  - 有界阻塞队列 ArrayBlockingQueue
  - 无界阻塞队列 LinkedBlockingQueue
  - 最多只有一个同步元素的队列 SynchronousQueue
- 优先队列 PriorityBlockingQueue
- threadFactory：线程工厂（给线程取名字）
- handler：拒绝策略

**工作方式：**

1. 线程池中刚开始没有线程，当一个任务提交给线程池后，线程池会创建一个新线程来执行任务。
2. 当线程数达到 corePoolSize 并没有线程空闲，这时再加入任务，新加的任务会被加入 workQueue 队列排 队，直到有空闲的线程。
3. 如果队列选择了有界队列，那么任务超过了队列大小时，会创建 maximumPoolSize - corePoolSize 数目的线 程来救急。
4. 如果线程到达 maximumPoolSize 仍然有新任务这时会执行拒绝策略。拒绝策略 jdk 提供了 下面的前 4 种实现，其它著名框架也提供了实现
   1. ThreadPoolExecutor.AbortPolicy 让调用者抛出RejectedExecutionException 异常，这是默认策略
   2. ThreadPoolExecutor.CallerRunsPolicy 让调用者运行任务
   3. ThreadPoolExecutor.DiscardPolicy 放弃本次任务
   4. ThreadPoolExecutor.DiscardOldestPolicy 放弃队列中最早的任务，本任务取而代之
   5. Dubbo 的实现，在抛出 RejectedExecutionException 异常之前会记录日志，并 dump 线程栈信息，方 便定位问题
   6. Netty 的实现，是创建一个新线程来执行任务
   7. ActiveMQ 的实现，带超时等待（60s）尝试放入队列，类似我们之前自定义的拒绝策略
   8. PinPoint 的实现，它使用了一个拒绝策略链，会逐一尝试策略链中每种拒绝策略
5. 当高峰过去后，超过 corePoolSize 的救急线程如果一段时间没有任务做，需要结束节省资源，这个时间由 keepAliveTime 和 unit 来控制。

jdk 线程池的拒绝策略结构图如下：
![在这里插入图片描述](../笔记图片/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70-17100938714301.png)
根据这个构造方法，JDK Executors 类中提供了众多工厂方法来创建各种用途的线程池

#### 3）newFixedThreadPool

这个是 Executors 类提供的静态的工厂方法来创建线程池！Executors 是 Executor 框架的工具类，newFixedThreadPool 创建的是固定大小的线程池。实现代码如下：

```java
		// 创建大小为 2 的固定线程池, 自定义线程名称
        ExecutorService executorService = Executors.newFixedThreadPool(2, new ThreadFactory() {
            private AtomicInteger atomicInteger = new AtomicInteger(1);
            @Override
            public Thread newThread(Runnable r) {
                return new Thread(r, "my_thread_" + atomicInteger.getAndIncrement());
            }
        });
        // 开 3 个线程, 线程池大小为 2 , 第三个线程执行时, 如果前两个线程任务没执行完, 会加入任务队列.
        executorService.execute(() -> {
            log.info("1");
        });
        executorService.execute(() -> {
            log.info("2");
        });
        executorService.execute(() -> {
            log.info("3");
        });
123456789101112131415161718
```

然后我再看看 Executors 类 使用 newFixedThreadPool 如何创建线程的，源码如下：

```java
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
12345678910111213
```

通过源码可以看到 new ThreadPoolExecutor(xxx) 方法其实是是调用了之前说的完整参数的构造方法，创建的是固定的线程数，使用了默认的线程工厂和拒绝策略。
**特点：**

- 核心线程数 == 最大线程数（没有救急线程被创建），因此也无需超时时间
- 阻塞队列是无界的（LinkedBlockingQueue），可以放任意数量的任务
- 适用于任务量已知，相对耗时的任务

#### 4）newCachedThreadPool

```java
    ExecutorService executorService = Executors.newCachedThreadPool();
	public static ExecutorService newCachedThreadPool() {
	 return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
	 60L, TimeUnit.SECONDS,
	 new SynchronousQueue<Runnable>());
	}
123456
```

特点

- 核心线程数是 0， 最大线程数是 Integer.MAX_VALUE，救急线程的空闲生存时间是 60s，意味着
  - 全部都是救急线程（60s 后没有任务就回收）
  - 救急线程可以无限创建
- 队列采用了 SynchronousQueue 实现特点是，它没有容量，没有线程来取是放不进去的（一手交钱、一手交 货）SynchronousQueue
- 整个线程池表现为线程数会根据任务量不断增长，没有上限，当任务执行完毕，空闲 1分钟后释放线程。
- 适合任务数比较密集，但每个任务执行时间较短的情况

#### 5）newSingleThreadExecutor

```java
public static ExecutorService newSingleThreadExecutor() {
 return new FinalizableDelegatedExecutorService
 (new ThreadPoolExecutor(1, 1,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>()));
}
1234
```

**使用场景：**

1. 希望多个任务排队执行。线程数固定为 1，任务数多于 1 时，会放入无界队列排队。任务执行完毕，这唯一的线程也不会被释放。
2. newSingleThreadExecutor 和 newFixedThreadPool 区别：
   1. 和自己创建单线程执行任务的区别：自己创建一个单线程串行执行任务，如果任务执行失败而终止那么没有任何补救措施，而线程池还会新建一个线程，保证池的正常工作
   2. Executors.newSingleThreadExecutor() 线程个数始终为 1 ，不能修改
      FinalizableDelegatedExecutorService 应用的是装饰器模式，只对外暴露了 ExecutorService 接口，因 此不能调用 ThreadPoolExecutor 中特有的方法
   3. 和Executors.newFixedThreadPool(1) 初始时为1时的区别：Executors.newFixedThreadPool(1) 初始时为1，以后还可以修改，对外暴露的是 ThreadPoolExecutor 对象，可以强转后调用 setCorePoolSize 等方法进行修改

**注意，Executors 返回线程池对象的弊端如下：**

- FixedThreadPool 和 SingleThreadExecutor ： 允许请求的队列长度为 Integer.MAX_VALUE,可能堆积大量的请求，从而导致 OOM。
- CachedThreadPool 和 ScheduledThreadPool ： 允许创建的线程数量为 Integer.MAX_VALUE ，可能会创建大量线程，从而导致 OOM。

说白了就是：使用有界队列，控制线程创建数量。
除了避免 OOM 的原因之外，不推荐使用 Executors提供的两种快捷的线程池的原因还有：

1. 实际使用中需要根据自己机器的性能、业务场景来手动配置线程池的参数比如核心线程数、使用的任务队列、饱和策略等等。
2. 我们应该显示地给我们的线程池命名，这样有助于我们定位问题。

#### 6）提交任务

```java
// 执行任务
void execute(Runnable command);
// 提交任务 task，用返回值 Future 获得任务执行结果，Future的原理就是利用我们之前讲到的保护性暂停模式来接受返回结果的，主线程可以执行 FutureTask.get()方法来等待任务执行完成
<T> Future<T> submit(Callable<T> task);
// 提交 tasks 中所有任务
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
 throws InterruptedException;
// 提交 tasks 中所有任务，带超时时间
<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
 long timeout, TimeUnit unit)
 throws InterruptedException;
// 提交 tasks 中所有任务，哪个任务先成功执行完毕，返回此任务执行结果，其它任务取消
<T> T invokeAny(Collection<? extends Callable<T>> tasks)
 throws InterruptedException, ExecutionException;
// 提交 tasks 中所有任务，哪个任务先成功执行完毕，返回此任务执行结果，其它任务取消，带超时时间
<T> T invokeAny(Collection<? extends Callable<T>> tasks,
 long timeout, TimeUnit unit)
 throws InterruptedException, ExecutionException, TimeoutException;
123456789101112131415161718
```

#### 7）关闭线程池

**shutdown**

```java
	/*
	  线程池状态变为 SHUTDOWN
	- 不会接收新任务
	- 但已提交任务会执行完，包括等待队列里面的
	- 此方法不会阻塞调用线程的执行
	*/
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 修改线程池状态
            advanceRunState(SHUTDOWN);
            // 仅会打断空闲线程
            interruptIdleWorkers();
            onShutdown(); // 扩展点 ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        // 尝试终结(没有运行的线程可以立刻终结)
        tryTerminate();
    }
12345678910111213141516171819202122
```

**shutdownNow**

```java
    /*
	线程池状态变为 STOP
	- 不会接收新任务
	- 会将队列中的任务返回
	- 并用 interrupt 的方式中断正在执行的任务
	*/
    public List<Runnable> shutdownNow() {

        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 修改线程池状态
            advanceRunState(STOP);
            // 打断所有线程
            interruptWorkers();
            // 获取队列中剩余任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        // 尝试终结
        tryTerminate();
        return tasks;
    }
1234567891011121314151617181920212223242526
```

**其它方法**

```java
// 不在 RUNNING 状态的线程池，此方法就返回 true
boolean isShutdown();
// 线程池状态是否是 TERMINATED
boolean isTerminated();
// 调用 shutdown 后，由于调用使线程结束线程的方法是异步的并不会等待所有任务运行结束就返回，因此如果它想在线程池 TERMINATED 后做些其它事情，可以利用此方法等待
boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
123456
```

#### 8）任务调度线程池

在『任务调度线程池』功能加入之前，可以使用 java.util.Timer 来实现定时功能，Timer 的优点在于简单易用，但 由于所有任务都是由同一个线程来调度，因此所有任务都是串行执行的，同一时间只能有一个任务在执行，前一个 任务的延迟或异常都将会影响到之后的任务。
使用 ScheduledExecutorService 改写：

- 整个线程池表现为：线程数固定，任务数多于线程数时，会放入无界队列排队。任务执行完毕，这些线程也不会被释放。用来执行延迟或反复执行的任务。
- ScheduledExecutorService 中 scheduleAtFixedRate 方法的使用，是 一段时间 的 期间。
- ScheduledExecutorService 中 scheduleWithFixedDelay 方法的使用，是 一段时间 的 间隔。

#### 9）正确处理执行任务异常

可以发现，如果线程池中的线程执行任务时，如果任务抛出了异常，默认是中断执行该任务而不是抛出异常或者打印异常信息。
方法1：主动捉异常

```java
	ExecutorService pool = Executors.newFixedThreadPool(1);
	pool.submit(() -> {
		 try {
			 log.debug("task1");
			 int i = 1 / 0;
		 } catch (Exception e) {
		 	log.error("error:", e);
		 }
	});
123456789
```

方法2：使用 Future，错误信息都被封装进submit方法的返回方法中

```java
	ExecutorService pool = Executors.newFixedThreadPool(1);
	Future<Boolean> f = pool.submit(() -> {
		 log.debug("task1");
		 int i = 1 / 0;
		 return true;
	});
	log.debug("result:{}", f.get());
1234567
```

#### 10）Tomcat 线程池

![在这里插入图片描述](../笔记图片/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70-17100938714302.png)

- LimitLatch 用来限流，可以控制最大连接个数，类似 J.U.C 中的 Semaphore 后面再讲
- Acceptor 只负责【接收新的 socket 连接】
- Poller 只负责监听 socket channel 是否有【可读的 I/O 事件】
- 一旦可读，封装一个任务对象（socketProcessor），提交给 Executor 线程池处理
- Executor 线程池中的工作线程最终负责【处理请求】

Tomcat 线程池扩展了 ThreadPoolExecutor，行为稍有不同，如果总线程数达到 maximumPoolSize，这时不会立刻抛 RejectedExecutionException 异常，而是再次尝试将任务放入队列，如果还失败，才抛出 RejectedExecutionException 异常。
源码 tomcat-7.0.42

```java
  public void execute(Runnable command, long timeout, TimeUnit unit) {
        submittedCount.incrementAndGet();
        try {
            super.execute(command);
        } catch (RejectedExecutionException rx) {
            if (super.getQueue() instanceof TaskQueue) {
                final TaskQueue queue = (TaskQueue)super.getQueue();
                try {
                    // 使任务从新进入阻塞队列
                    if (!queue.force(command, timeout, unit)) {
                        submittedCount.decrementAndGet();
                        throw new RejectedExecutionException("Queue capacity is full.");
                    }
                } catch (InterruptedException x) {
                    submittedCount.decrementAndGet();
                    Thread.interrupted();
                    throw new RejectedExecutionException(x);
                }
            } else {
                submittedCount.decrementAndGet();
                throw rx;
            }
        }
    }

    public boolean force(Runnable o, long timeout, TimeUnit unit) throws InterruptedException {
        if ( parent.isShutdown() )
            throw new RejectedExecutionException(
                    "Executor not running, can't force a command into the queue"
            );
        return super.offer(o,timeout,unit); //forces the item onto the queue, to be used if the task
        is rejected
    }
123456789101112131415161718192021222324252627282930313233
```

**Connector 配置如下：**
![在这里插入图片描述](../笔记图片/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70-17100938714313.png)**Executor 线程池配置如下：**
![在这里插入图片描述](../笔记图片/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70-17100938714314.png)
可以看到该线程池实现的是一个无界的队列，所以说是不是执行任务的线程数大于了核心线程数，都会添加到阻塞队列中，那么救急线程是不是就不会用到呢，其实不是，分析如下图：![在这里插入图片描述](../笔记图片/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl81MDI4MDU3Ng==,size_16,color_FFFFFF,t_70-17100938714315.png)
如图所示，当添加新的任务时，如果提交的任务大于核心线程数小于最大线程数就创建救急线程，否则就加入任务队列中。

#### 异步模式之工作线程

**定义：**
让有限的工作线程（Worker Thread）来轮流异步处理无限多的任务。也可以将其归类为分工模式，它的典型实现就是线程池，也体现了经典设计模式中的享元模式。
**例如：**
海底捞的服务员（线程），轮流处理每位客人的点餐（任务），如果为每位客人都配一名专属的服务员，那 么成本就太高了（对比另一种多线程设计模式：Thread-Per-Message） 注意，不同任务类型应该使用不同的线程池，这样能够避免饥饿，并能提升效率 例如，如果一个餐馆的工人既要招呼客人（任务类型A），又要到后厨做菜（任务类型B）显然效率不咋地，分成 服务员（线程池A）与厨师（线程池B）更为合理，当然你能想到更细致的分工。
**饥饿：**
固定大小线程池会有饥饿现象

1. 两个工人是同一个线程池中的两个线程
2. 他们要做的事情是：为客人点餐和到后厨做菜，这是两个阶段的工作
   1. 客人点餐：必须先点完餐，等菜做好，上菜，在此期间处理点餐的工人必须等待
   2. 后厨做菜：没啥说的，做就是了
3. 比如工人 A 处理了点餐任务，接下来它要等着 工人 B 把菜做好，然后上菜，他俩也配合的蛮好 但现在同时来了两个客人，这个时候工人 A 和工人 B 都去处理点餐了，这时没人做饭了，这就是饥饿。

解决方法可以增加线程池的大小，不过不是根本解决方案，还是前面提到的，不同的任务类型，采用不同的线程池。实现代码如下：

```java
/**
 * 异步模式之工作线程
 */
@Slf4j(topic = "c.Code_07_StarvationTest")
public class Code_07_StarvationTest {

    public static List<String> list = new ArrayList<>(Arrays.asList("宫保鸡丁", "青椒肉丝", "千张肉丝"));
    public static Random random = new Random();
    public static String cooking() {
        return list.get(random.nextInt(list.size()));
    }

    public static void main(String[] args) {
        // 演示饥饿现象
//        ExecutorService executorService = Executors.newFixedThreadPool(2);
//        test1(executorService);

        // 解决
        ExecutorService cookPool = Executors.newFixedThreadPool(1);
        ExecutorService waiterPool = Executors.newFixedThreadPool(1);

        waiterPool.execute(() -> {
            log.info("处理点餐");
            Future<String> f = cookPool.submit(() -> {
                log.info("做菜");

                return cooking();
            });
            try {
                log.info("上菜 {} ", f.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });

        waiterPool.execute(() -> {
            log.info("处理点餐");
            Future<String> f2 = cookPool.submit(() -> {
                log.info("做菜");
                return cooking();
            });
            try {
                log.info("上菜 {} ", f2.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
    }

    public static void test1( ExecutorService executorService) {
        executorService.execute(() -> {
            log.info("处理点餐");
            Future<String> f = executorService.submit(() -> {
                log.info("做菜");
                return cooking();
            });
            try {
                log.info("上菜 {} ", f.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
        executorService.execute(() -> {
            log.info("处理点餐");
            Future<String> f2 = executorService.submit(() -> {
                log.info("做菜");
                return cooking();
            });
            try {
                log.info("上菜 {} ", f2.get());
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
    }

}


123456789101112131415161718192021222324252627282930313233343536373839404142434445464748495051525354555657585960616263646566676869707172737475767778798081828384858687
```

**创建多大的线程池合适?**
过小会导致程序不能充分地利用系统资源、容易导致饥饿，过大会导致更多的线程上下文切换，占用更多内存，

1. CPU 密集型运算 通常采用 cpu 核数 + 1 能够实现最优的 CPU 利用率，+1 是保证当线程由于页缺失故障（操作系统）或其它原因导致暂停时，额外的这个线程就能顶上去，保证 CPU 时钟周期不被浪费
2. I/O 密集型运算 CPU 不总是处于繁忙状态，例如，当你执行业务计算时，这时候会使用 CPU 资源，但当你执行 I/O 操作时、远程 RPC 调用时，包括进行数据库操作时，这时候 CPU 就闲下来了，你可以利用多线程提高它的利用率。
   1. 经验公式如下：
   2. 线程数 = 核数 * 期望 CPU 利用率 * 总时间(CPU计算时间+等待时间) / CPU 计算时间
   3. 例如 4 核 CPU 计算时间是 50% ，其它等待时间是 50%，期望 cpu 被 100% 利用，套用公式 4 * 100% * 100% / 50% = 8
   4. 例如 4 核 CPU 计算时间是 10% ，其它等待时间是 90%，期望 cpu 被 100% 利用，套用公式 4 * 100% * 100% / 10% = 40

#### 应用之定时任务

使用 newScheduledThreadPool 中的 scheduleAtFixedRate 这个方法可以执行定时任务。代码如下：

```java
 public static void main(String[] args) {
        // 获取当前时间
        LocalDateTime now = LocalDateTime.now();
        System.out.println(now);
        // 获取每周四晚时间
        LocalDateTime time = now.withHour(18).withMinute(0).withSecond(0).withNano(0).with(DayOfWeek.THURSDAY);
        if(now.compareTo(time) > 0) {
            time = time.plusWeeks(1);
        }

        long initalDelay = Duration.between(now, time).toMillis();
        // 一周的时间
        long period = 1000 * 60 * 60 * 24 * 7;
        // initalDelay 表示当前时间与周四的时间差, period 一周的间隔时间。
        ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
        // 创建一个定时任务, 每周四 18:00:00 执行。
        executorService.scheduleAtFixedRate(() -> {
            System.out.println("running");
        }, initalDelay, period, TimeUnit.MILLISECONDS);
    }
1234567891011121314151617181920
```

### 4、Fork/Join

参考如下文章：
[Fork/Join框架原理解析](https://blog.csdn.net/tyrroo/article/details/81483608)
[Fork/Join框架学习](https://www.jianshu.com/p/42e9cd16f705)https://www.jianshu.com/p/42e9cd16f705)