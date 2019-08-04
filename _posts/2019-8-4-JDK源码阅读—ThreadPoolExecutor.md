---
layout:     post
title:      JDK源码阅读—ThreadPoolExecutor
subtitle:   
date:       2019-8-4
author:     BY KiloMeter
header-img: img/2019-8-4-JDK源码阅读—ThreadPoolExecutor/65.jpg
catalog: true
tags:
    - 源码阅读
---

ThreadPoolExecutor是Java中的线程池

ThreadPoolExecutor继承了AbstractExecutorService，而AbstractExecutorService实现了ExecutorService接口，ExecutorService继承了Executor，关于这几个类之间的关系，可以看下[深入理解 Java 线程池：ThreadPoolExecutor](https://juejin.im/entry/58fada5d570c350058d3aaad)

下面就来看下ThreadPoolExecutor的源代码

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
```

这几个是ThreadPoolExecutor内部的几个字段。

`ctl`是对线程池的运行状态和线程池中有效线程的数量进行控制的一个字段， 它包含两部分的信息: 线程池的运行状态 (runState) 和线程池内有效线程的数量 (workerCount)，这里可以看到，使用了Integer类型来保存，高3位保存runState，低29位保存workerCount。COUNT_BITS 就是29，CAPACITY就是1左移29位减1（29个1），这个常量表示workerCount的上限值，大约是5亿。

下面的几个字段指的是线程池的**五种运行状态**

* **RUNNING**：该状态的线程池可以接受新提交的任务，并且也能处理阻塞队列中的任务；
* **SHUTDOWN**：关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务。在线程池处于 RUNNING 状态时，调用 shutdown()方法会使线程池进入到该状态。（finalize() 方法在执行过程中也会调用shutdown()方法进入该状态）；
* **STOP**：不能接受新任务，也不处理队列中的任务，会中断正在处理任务的线程。在线程池处于 RUNNING 或 SHUTDOWN 状态时，调用 shutdownNow() 方法会使线程池进入到该状态；
* **TIDYING**：如果所有的任务都已终止了，workerCount (有效线程数) 为0，线程池进入该状态后会调用 terminated() 方法进入TERMINATED 状态。
* **TERMINATED**：在terminated() 方法执行完后进入该状态，默认terminated()方法中什么也没有做。

下面是线程池内部的核心变量，这些参数将结合构造方法来解释

```java
private final BlockingQueue<Runnable> workQueue;
private final ReentrantLock mainLock = new ReentrantLock();
private final HashSet<Worker> workers = new HashSet<Worker>();
private final Condition termination = mainLock.newCondition();
private int largestPoolSize;
private long completedTaskCount;
private volatile ThreadFactory threadFactory;
private volatile RejectedExecutionHandler handler;
private volatile long keepAliveTime;
private volatile boolean allowCoreThreadTimeOut;
private volatile int corePoolSize;
private volatile int maximumPoolSize;


public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
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
```

**corePoolSize**：核心线程数，当任务提交时，会执行以下判断：

1. 如果运行的线程少于 corePoolSize，则创建新线程来处理任务，即使线程池中的其他线程是空闲的；
2. 如果线程池中的线程数量大于等于 corePoolSize 且小于 maximumPoolSize，则只有当workQueue满时才创建新的线程去处理任务；
3. 如果设置的corePoolSize 和 maximumPoolSize相同，则创建的线程池的大小是固定的，这时如果有新任务提交，若workQueue未满，则将请求放入workQueue中，等待有空闲的线程去从workQueue中取任务并处理；
4. 如果运行的线程数量大于等于maximumPoolSize，这时如果workQueue已经满了，则通过handler所指定的策略来处理任务；

**maximumPoolSize**：最大线程数

**keepAliveTime**：线程池维护线程所允许的空闲时间。当线程池中的线程数量大于corePoolSize的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了keepAliveTime；

**unit**：keepAliveTime的时间单位

**workQueue**：等待队列，当任务提交时，如果线程池中的线程数量大于等于corePoolSize的时候，把该任务封装成一个Worker对象放入等待队列；

**threadFactory**：它是ThreadFactory类型的变量，用来创建新线程。默认使用Executors.defaultThreadFactory() 来创建线程。使用默认的ThreadFactory来创建线程时，会使新创建的线程具有相同的NORM_PRIORITY优先级并且是非守护线程，同时也设置了线程的名称。

**handler**：它是RejectedExecutionHandler类型的变量，表示线程池的饱和策略。如果阻塞队列满了并且没有空闲的线程，这时如果继续提交任务，就需要采取一种策略处理该任务。线程池提供了4种策略：

* AbortPolicy：直接抛出异常，这是默认策略；
* CallerRunsPolicy：用调用者所在的线程来执行任务；
* DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
* DiscardPolicy：直接丢弃任务；



```java
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        //ctl记录线程池运行状态和线程数量
        int c = ctl.get();
        //如果线程数量少于corePoolSize，则新建线程并执行该任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        //如果是正在运行状态且任务成功添加到队列中
        if (isRunning(c) && workQueue.offer(command)) {
            //重新获得ctl值
            int recheck = ctl.get();
            //再次判断线程池的运行状态，如果不是运行状态，由于之前已经把command添加到workQueue中了，
            //这时需要移除该command并通过handler使用拒绝策略对该任务进行处理
            if (! isRunning(recheck) && remove(command))
                reject(command);
            //如果线程池中的线程数量为0，
            //这里传入的参数表示：
            //1. 第一个参数为null，表示在线程池中创建一个线程，但不去启动；
            //2. 第二个参数为false，将线程池的有限线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断；
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //执行到这里，说明线程池不是处于RUNNING状态或者线程池是RUNNING状态，但workerCount >= corePoolSize并且workQueue已满。
        //这两种情况均使用拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```

addWorker方法的主要工作是在线程池中创建一个新的线程并执行，firstTask参数 用于指定新增的线程执行的第一个任务，core参数为true表示在新增线程时会判断当前活动线程数是否少于corePoolSize，false表示新增线程前需要判断当前活动线程数是否少于maximumPoolSize

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            //rs>=SHUTDOWN时，不会再接收任务
            //接着判断以下3个条件，只要有1个不满足，则返回false：
            //1. rs == SHUTDOWN，这时表示关闭状态，不再接受新提交的任务，但却可以继续处理阻塞队列中已保存的任务
            //2. firsTask为空
            //3. 阻塞队列不为空
            
            //如果rs为shutdown，不会再接收新任务，因此如果first不为空，则返回false
            //如果first为null，并且workQueue也为null，意味着没有任务了，也返回false
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                //如果当前线程大于容量，即2^29-1，返回false
                //或者根据core来进行判断，core是方法的第二个参数，如果core为true，则将当前线程数和
                //corePoolSize比较，否则和maximumPoolSize比较
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 尝试增加workerCount，如果成功，则跳出第一个for循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 如果增加workerCount失败，则重新获取ctl的值
                c = ctl.get();
                // 如果当前的运行状态不等于rs，说明状态已被改变，返回第一个for循环继续执行
                if (runStateOf(c) != rs)
                    continue retry;
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            // 根据firstTask来创建Worker对象
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    int rs = runStateOf(ctl.get());
                    //rs<SHUTDOWN以为着线程池是RUNNING状态
                    //或者rs是处于SHUTDOWN状态并且first为null，则向线程池中添加线程，因为在shutdown状态时不会添加新任务，但是仍会处理workQueue中的任务
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
                    //启动线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

线程池中的每一个线程被封装成一个Worker对象，ThreadPool维护的其实就是一组Worker对象

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {

        private static final long serialVersionUID = 6138294804551838833L;
        //thread是在调用构造方法时通过ThreadFactory来创建的线程，是用来处理任务的线程。
        final Thread thread;
        //firstTask用来保存传入的任务
        Runnable firstTask;

        volatile long completedTasks;

        Worker(Runnable firstTask) {
            setState(-1);
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        public void run() {
            runWorker(this);
        }

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }
```

线程池内容有点多，暂时先说到这，剩下的内容之后再补充。