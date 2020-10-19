
Java提供了几种便捷的方法创建线程池，通过这些内置的api就能够很轻松的创建线程池。在`java.util.concurrent`包中的`Executors`类，其中的静态方法就是用来创建线程池的：

* newFixedThreadPool()：创建一个固定线程数量的线程池，而且线程池中的任务全部执行完成后，空闲的线程也不会被关闭。
* newSingleThreadExecutor()：创建一个只有一个线程的线程池，空闲时也不会被关闭。
* newCachedThreadPool()：创建一个可缓存的线程池，线程的数量为`Integer.MAX_VALUE`，空闲线程会临时缓存下来，线程会等待`60s`还是没有任务加入的话就会被关闭。

`Executors`类中还有一些创建线程池的方法（jdk8新加的），但是现在这个触极到我的知识盲区了~~

![](https://arch-digest.oss-cn-shanghai.aliyuncs.com/thread-pool/14301602992990_.pic_hd.jpg)


上面那几个方法，其实都是创建了一个`ThreadPoolExecutor`对象作为返回值，要搞清楚线程池的原理主要还是要分析`ThreadPoolExecutor`这个类。

`ThreadPoolExecutor`的构造方法：

```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
      ...
    }
```
 
`ThreadPoolExecutor`的构造方法包含以下几个参数：

 
 * corePoolSize： 核心线程数量，常驻线程池中的线程，即时线程池中没有任务可执行，也不会被关闭。
 * maximumPoolSize：最大线程数量
 * keepAliveTime：空闲线程存活时间
 * unit： 空闲线程存活时间的单位
 * workQueue：工作队列，线程池一下忙不过来，那新来的任务就需要排队，排除中的任务就会放在workQueue中
 * threadFactory：线程工厂，创建线程用的
 * handler：`RejectedExecutionHandler`实例用于在线程池中没有空闲线程能够执行任务，并且`workQueue`中也容不下任务时拒绝任务时的策略。

`ThreadPoolExecutor`中的线程统称为工作线程，但有一个小概念是`核心线程`，核心线程由参数`corePoolSize`指定，如`corePoolSize`设置5，那线程池中就会有5条线程常驻线程池中，不会被回收掉，但是也会有例外，如果`allowCoreThreadTimeOut`为`true`空闲一段时间后，也会被关闭。

### 线程的状态和工作线程数量

线程中的状态和工作线程和数量都是由`ctl`表示，是一个`AtomicInteger`类型的属性：

```
 private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

ctl的高四位为线程的状态，其他位数为工作线程的数量，所以线程中最大的工作线程数量为`(2^29)-1`。


线程池中的状态有五种：

* RUNNING：接收新的任务和处理队列中的任务
* SHUTDOWN：不能新增任务，但是会继续处理已经添加的任务
* STOP：不能新增任务，不会继续处理已经添加任务
* TIDYING：所有的任务已经被终止，工作线程为0
* TERMINATED：terminated()方法执行完成

状态码的定义如下：

```
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
    


### 创建线程池

如果有面试官问：如何正确的创建线程池？千万不要说使用`Executors`创建线程，虽然`Executors`能很方便的创建线程池，但是他提供的静态创建方法会有一些坑。

**主要的原因是：`maximumPoolSize`和`workQueue`这两个参数**

`Executors`静态方法在创建线程池时，如果`maximumPoolSize`设置为`Integer.MAX_VALUE`，这样会导致线程池可以一直要以接收运行任务，可能导致cpu负载过高。

`workQueue`是一个阻塞队列的实例，用于放置正在等待执行的任务。如果在创建线程种时`workQueue`实例没有指定任务的容量，那么等待队列中可以一直添加任务，极有可能导致`oom`。

所以创建线程，**最好是根据线程池的用途，然后自己创建线程**。


### 添加任务

调用线程池的`execute`并不是立即执行任务，线程池内部用经过一顿操作，如：判断核心线程数、是否需要添加到等待队列中。

下来的代码是`execute`的源码，代码很简洁只有2个`if`语句：


```
 public void execute(Runnable command) {
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

 

1. 第一个if，如果当前线程池中的工作线程数量小于`corePoolSize`，直接创建一个工作线程执行任务
2. 第二个if，当线程池处于运行状态，调用`workQueue.offer(command)`方法将任务添加到`workQueue`，否则调用`addWorker(command, false)`尝试去添加一个工作线程。

 

整理了一张图，把线程池分为三部分`Core Worker`、`Worker`、`workQueue`：

![](https://arch-digest.oss-cn-shanghai.aliyuncs.com/thread-pool/14311602993000_.pic.jpg)


换一种说法，在调用`execute`方法时，任务首先会放在`Core Worker`内，然后才是`workQueue`，最后才会考虑`Worker`。

这样做的原因可以保证`Core Worker`中的任务执行完成后，能立即从`workQueue`获取下一个任务，而不需要启动别的工作线程，用最少的工作线程办更多的事。
 

### 创建工作线程

在`execute`方法中，有三个地方调用了`addWorker`。`addWorker`方法可以分为二部分：

1. 增加工作线程数量
2. 启动工作线程


`addWorker`的方法签名如下：

```
private boolean addWorker(Runnable firstTask, boolean core) 
```

* **firstTask**：第一个运行的任务，可以为空。如果为空任务会从`workQueue`中获取。
* **core**： 是否是核心工作线程
 


#### 增加工作线程数量
 

```
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

     		....

        for (;;) {
            int wc = workerCountOf(c);
            
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
                
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
```

上面代码省略了一部分代码，主要代码都在`for`循环中，利用`CAS`锁，安全的完成线程池状态的检查与增加工作线程的数量。其中的`compareAndIncrementWorkerCount(c)`调用就是将工作线程数量+1。

#### 启动工作线程

增加工作线程的数量后，紧接着就会启动工作线程：

```
boolean workerStarted = false;
boolean workerAdded = false;
Worker w = null;
try {
    w = new Worker(firstTask);
    final Thread t = w.thread;
    if (t != null) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // Recheck while holding lock.
            // Back out on ThreadFactory failure or if
            // shut down before lock acquired.
            int rs = runStateOf(ctl.get());

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
            t.start();
            workerStarted = true;
        }
    }
} finally {
    if (! workerStarted)
        addWorkerFailed(w);
}
```

启动工作线程的流程：

* 创建一个`Worker`实例, `Worker`构造方法会使用`ThreadFactory`创建一个线程
 
```
w = new Worker(firstTask);
final Thread t = w.thread;
```

就不说`Worker`类的实现了，直接给出构造方法来细品：

```
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```

* 如果线程池状态是在运行中，或者已经关闭，但工作线程要从`workQueue`中获取任务，才能添加工作线程

```
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
```

**注意：**：当线程池处于`SHUTDOWN`状态时，它不能接收新的任务，但是可以继续执行未完成的任务。任务是否从`workQueue`中获取，是根据`firstTask`判断，每个`Worker`实例都有一个`firstTask `属性，如果这个值为`null`，工作线程启动的时候就会从`workQueue`中获取任务，否则会执行`firstTask `。

* 启动线程

调用线程的`start`方法，启动线程。

```
 if (workerAdded) {
    t.start();
    workerStarted = true;
}
```


### 执行任务

回过头来看一个`Worker`类的定义：

```
 private final class Worker extends AbstractQueuedSynchronizer
	implements Runnable{
		Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
      ...
}
```

`Worker`类实现了`Runnable`接口，同时在构造方法中会将`this`传递给线程，到这里你就知道了`Worker`实例中有`run`方法，它会在线程启动后执行：

```
public void run() {
   runWorker(this);
}
```

`run`方法内部接着调用`runWorker`方法运行任务，在这里才是真正的开始运行任务了：

```
 final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
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
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

* 获取任务

首先将`firstTask`传递给`task`临时变量：

```
Runnable task = w.firstTask;

```

然后循环检查`task`或者从`workQueue`中获取任务：

```
while (task != null || (task = getTask()) != null) {
	...
}
```

`getTask()`稍后再做分析。


* 运行任务

去掉一些状态检查、异常捕获、和勾子方法调用后，保留最重要的调用`task.run()`：

```
 while (task != null || (task = getTask()) != null) {
     ...          
     task.run();
     ...
 }
```

`task`其实就是通过调用`execute`方法传递进来的`Runnable`实例，也就是你的任务。只不过它可能保存在`Worker.firstTask`中，或者在`workQueue`中，保存在哪里在前面的`任务添加顺序`中已经说明。


### 从workQueue中获取任务

试想一下如果每个任务执行完成，就关闭掉一个线程那有多浪费资源，这样使用线程池也没有多大的意义。所以线程的主要的功能就是线程复用，一旦任务执行完成直接去获取下一个任务，或者挂起线程等待下一个提交的任务，然后等待一段时间后还是没有任务提交，然后才考虑是否关闭部分空闲的线程。


`runWorker`中会循环的获取任务：

```
 while (task != null || (task = getTask()) != null) {
     ...          
     task.run();
     ...
 }
```

上面的代码`getTask()`就是从`workQueue`中获取任务：

```
  private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
        			...
             int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
				 ...
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
```
 
 获取任务的时候会有两种方式：
 
 1. 超时等待获取任务
 2. 一直等待任务，直到有新任务
 
如果`allowCoreThreadTimeOut`为`true`，`corePoolSize`指定的核心线程数量会被忽略，直接使用 ` workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)` 获取任务，否则的话会根据当前工作线程的数量，如果`wc > corePoolSize`为`false`则当前会被认为是核心线程，调用`workQueue.take()`一直等待任务。


### 工作线程的关闭

还是在`runWorker`方法中：

```
 final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
           
                  task.run();
                       
            }
                 
            completedAbruptly = false;   
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }

```

* completedAbruptly变量：标记当前工作线程是正常执行完成，还是异常完成的。completedAbruptly为`false`可以确定线程池中没有可执行的任务了。


上面代码是简洁后的代码，一个`while`循环保证不间断的获取任务，没有任务可以执行（task为null）退出循环，最后再才会调用`processWorkerExit`方法：

```
 private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```

`processWorkerExit`接收一个`Worker`实例与`completedAbruptly`变量。processWorkerExit的大致工作流程：

* 判断当前工作线程是否异常完成，如果是直接减少工作线程的数量，简单的说就是校正一下工作线程的数量。
* 增加完成的任务数量，将`Worker`从`workers`中移除
* tryTerminate() 检查线程池状态，因为线程池可以延迟关闭，如果你调用`shutdown`方法后不会立即关闭，要等待所有的任务执行完成，所以这里调用tryTerminate()方法，尝试去调用`terminated`方法。

#### 工作线程完成策略

如果某个工作线程完成，线程池内部会判断是否需要重新启动一个：


```
//判断线程池状态
if (runStateLessThan(c, STOP)) {
    if (!completedAbruptly) {
    		//获取最小工作线程数量
        int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
        //如果最小工作线程数量为0，但是workQueue中还有任务，那重置最小工作线程数量1
        if (min == 0 && ! workQueue.isEmpty())
            min = 1;
        //如果当前工作线程数数量大于或等于最小工作线程数量，则不需要启动新的工作线程
        if (workerCountOf(c) >= min)
            return; // replacement not needed
    }
    
    //启动一个新的工作线程
    addWorker(null, false);
}
```

工作线程完成后有两种处理策略：

1. 对于异常完成的工作线程，直接启动一个新的替换
2. 对于正常完成的工作线程，判断当前工作线程是否足够，如果足够则不需要新启动工作线程

**注意：**这里的完成，表示工作线程的任务执行完成，`workQueue`中也没有任务可以获取了。


### 线程池的关闭

关闭线程池有可以通过`shutdown`方法：

```
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

`shutdown`方法，第一步就是先改变线程池的状态，调用`advanceRunState(SHUTDOWN)`方法，将线程池当前状态更改为`SHUTDOWN`，advanceRunState代码如下：

```
 private void advanceRunState(int targetState) {
        for (;;) {
            int c = ctl.get();
            if (runStateAtLeast(c, targetState) ||
                ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
                break;
        }
    }
```

然后立即调用`interruptIdleWorkers()`方法，`interruptIdleWorkers()`内部会调用它的重载方法`interruptIdleWorkers(boolean onlyOne)`同时onlyOne参数传递的`false`来关闭空闲的线程：

```
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}
    
private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```

以上代码会遍历`workers`中的`Worker`实例，然后调用线程的`interrupt()`方法。

#### 什么样的线程才是空闲工作线程？

前面提到过在`getTask()`中，线程从`workQueue`中获取任务时会阻塞，**被阻塞的线程就是空闲的**。

再次回到`getTask()`的代码中：

```
  private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
        
	          // Check if queue empty only if necessary.
	            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
	                decrementWorkerCount();
	                return null;
	            }
        			...
             int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
				 ...
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
```
再次分析`getTask()`中的代码中有一段捕获`InterruptedException`的代码块，interruptIdleWorkers方法中断线程后，`getTask()`会捕获中断异常，因为外面是一个`for`循环，随后代码走到判断线程池状态的地方：

```
if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
	                decrementWorkerCount();
	                return null;
}
```

上面的代码的会判断当前线程池状态，如果状态大于`STOP`或者状态等于`SHUTDOWN`并且`workQueue`为空时则返回`null`，`getTask()`返回空那么在`runWorker`中循环就会退出，当前工作线程的任务就完成了，可以退出了：

```
 final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
           
                  task.run();
                       
            }
                 
            completedAbruptly = false;   
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```


#### shutdownNow

除了shutdown方法能关闭线程池，还有`shutdownNow`也可以关闭线程池。它两的区别在于:

* `shutdownNow`会清空`workQueue`中的任务
* `shutdownNow`还会中止当前正在运行的任务
* `shutdownNow`会使线程进入`STOP`状态，而`shutdown()`是`SHUTDOWN`状态

```
public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```

上面代码基本流程：

*  advanceRunState(STOP): 使线程池进行`STOP`状态，与`shutdown()`中的一致 ，只是使用的状态码是`STOP`
*  interruptWorkers()： 与`shutdown()`中的一致
*  drainQueue(): 清空队列

####  任务是中止执行还是继续执行？

调用shutdownNow()后线程池处于`STOP`状态，紧接着所有的工作线程都会被调用`interrupt`方法，如果此时`runWorker`还在运行会发生什么？

在`runWorker`有一段代码，就是工作线程中止的重要代码：
  
```
 final void runWorker(Worker w) {
        ...
        
         while (task != null || (task = getTask()) != null) {
           
            		if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                    
                  task.run();
                       
            }
        ...
}

```

重点关注：

```
if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
```

这个if看起来有点难理解，理解下来大致意思是：如果线程池状态大于等于`STOP`，立即中断线程，否则清除线程的中断标记，也就是说当线程池状态为`RUNNING`和`SHUTDOWN`时，线程的中断标记会被清除（线程的中断代码在`interruptWorkers`方法中），可以继续执行任务。

以上代码执行完成后，紧接着就会调用`task.run()`方法，**这里面我们自己就可以根据线程的中断标记来判断任务是否被中断。**

### 总结

个人水平有限，文中如有错误，谢谢大家指正。

本文从线程池的源码入手，分析线程池的创建、添加任务、运行任务等流程，整个分析下来基本上大多数公司关于线程池面试的问题都可以回答得上来，当然还有一些小细节如：`Worker`类是继承`AQS`的，为什么这么做其实源码中都有一些苗头，`Worker`在运行时会锁住运行的代码块，而`shutdown`在关闭空闲的`Worker`时，首先就要去获取`Worker`的同步锁才能继续操作，这样才能安全的关闭工作线程。






 



 

















 
 
 