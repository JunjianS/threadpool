Java通过以下方式定义一个线程池，其中各参数的含义参考原理中的介绍

```
ThreadPoolExecutor commonExecutor = new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS,new ArrayBlockingQueue<Runnable>(30));
```

通过以下方式提交一个任务到线程池
```
commonExecutor.execute(new Thread() {
				public void run() {
					try {
						TimeUnit.SECONDS.sleep(1);
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					System.out.println("Number of completed tasks : " + commonExecutor.getCompletedTaskCount());
					System.out.println("Number of remaining tasks : " + commonExecutor.getQueue().size());
				}
			});
```
可以看到核心的方法就是execute，下面就从该方法入手，分析一下Java线程池的源码

```
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         * 如果当前线程数小于线程池核心大小，尝试调用addWorker方法启动新的线程
         * 并以command为改工作线程的第一个任务，
         * 
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         * 如果一个任务能够成功加入队列，那么我们仍然需要再次确认是否需要增加一个线程，
         * 或者线程池关闭当进入这个方法的时；所有再次确认线程池状态，当线程池停止时回滚任务如队列操作，
         * 或者线程池为空时启动一个新的额线程。
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         * 如果不能一个task不能加入队列，尝试增加一个新的线程。如果失败，也就是说线程池已经关闭或者饱和，
         * 就拒绝这个task
         */
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

下面我们继续看看addWorker是如何实现的：

```
private boolean addWorker(Runnable firstTask, boolean core) {
// java标签
retry:
    // 死循环
    for (;;) {
        int c = ctl.get();
        // 获取当前线程池状态
        int rs = runStateOf(c);
        // Check if queue empty only if necessary.
        // 这个逻辑判断有点绕可以改成
        // rs >= shutdown && (rs != shutdown || firstTask != null || workQueue.isEmpty())
        // 逻辑判断成立可以分为以下几种情况均不接受新任务
        // 1、rs > shutdown:--不接受新任务
        // 2、rs >= shutdown && firstTask != null:--不接受新任务
        // 3、rs >= shutdown && workQueue.isEmppty:--不接受新任务
        // 逻辑判断不成立
        // 1、rs==shutdown&&firstTask != null:此时不接受新任务，但是仍会执行队列中的任务
        // 2、rs==shotdown&&firstTask == null:会执行addWork(null,false)
        //  防止了SHUTDOWN状态下没有活动线程了，但是队列里还有任务没执行这种特殊情况。
        //  添加一个null任务是因为SHUTDOWN状态下，线程池不再接受新任务
        if (rs >= SHUTDOWN &&! (rs == SHUTDOWN && firstTask == null &&! workQueue.isEmpty()))
            return false;
        // 死循环
        // 如果线程池状态为RUNNING并且队列中还有需要执行的任务
        for (;;) {
            // 获取线程池中线程数量
            int wc = workerCountOf(c);
            // 如果超出容量或者最大线程池容量不在接受新任务
            if (wc >= CAPACITY || wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 线程安全增加工作线程数
            if (compareAndIncrementWorkerCount(c))
                // 跳出retry
                break retry;
            c = ctl.get();          // Re-read ctl
            // 如果线程池状态发生变化，重新循环
            if (runStateOf(c) != rs)
                continue retry;
        // else CAS failed due to workerCount change; retry inner loop
        }
    }
    // 走到这里说明工作线程数增加成功
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        final ReentrantLock mainLock = this.mainLock;
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            // 加锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int c = ctl.get();
                int rs = runStateOf(c);
                // RUNNING状态 || SHUTDONW状态下清理队列中剩余的任务
                if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                    // 检查线程状态
                    if (t.isAlive())                 
                        throw new IllegalThreadStateException();
                    // 将新启动的线程添加到线程池中
                    workers.add(w);
                    // 更新线程池线程数且不超过最大值
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            }
            finally {
                mainLock.unlock();
            }
                // 启动新添加的线程，这个线程首先执行firstTask，然后不停的从队列中取任务执行
            if (workerAdded) {
                //执行ThreadPoolExecutor的runWoker方法
                t.start();
                workerStarted = true;
            }
        }
    }
    finally {
        // 线程启动失败，则从wokers中移除w并递减wokerCount
        if (! workerStarted)
            // 递减wokerCount会触发tryTerminate方法
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

addWorker之后是runWorker,第一次启动会执行初始化传进来的任务firstTask；然后会从workQueue中取任务执行，如果队列为空则等待keepAliveTime这么长时间

```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 允许中断
    w.unlock();        
    boolean completedAbruptly = true;
    try {
        // 如果getTask返回null那么getTask中会将workerCount递减，如果异常了这个递减操作会在processWorkerExit中处理
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
                    thrown = x;
                    throw x;
                } catch (Error x) {
                    thrown = x;
                    throw x;
                } catch (Throwable x) {
                    thrown = x;
                    throw new Error(x);
                }
                finally {
                    afterExecute(task, thrown);
                }
            }
            finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    }
    finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

我们看下getTask是如何执行的

```
private Runnable getTask() {
    boolean timedOut = false;         
// 死循环
retry:
    for (;;) {
        // 获取线程池状态
        int c = ctl.get();
        int rs = runStateOf(c);
        // Check if queue empty only if necessary.
        // 1.rs > SHUTDOWN 所以rs至少等于STOP,这时不再处理队列中的任务
        // 2.rs = SHUTDOWN 所以rs>=STOP肯定不成立，这时还需要处理队列中的任务除非队列为空
        // 这两种情况都会返回null让runWoker退出while循环也就是当前线程结束了，所以必须要decrement
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            // 递减workerCount值
            decrementWorkerCount();
            return null;
        }
        // 标记从队列中取任务时是否设置超时时间
        boolean timed;         
        // 1.RUNING状态
        // 2.SHUTDOWN状态，但队列中还有任务需要执行
        for (;;) {
            int wc = workerCountOf(c);
            // 1.core thread允许被超时，那么超过corePoolSize的的线程必定有超时
            // 2.allowCoreThreadTimeOut == false && wc >
            // corePoolSize时，一般都是这种情况，core thread即使空闲也不会被回收，只要超过的线程才会
            timed = allowCoreThreadTimeOut || wc > corePoolSize;
            // 从addWorker可以看到一般wc不会大于maximumPoolSize，所以更关心后面半句的情形：
            // 1. timedOut == false 第一次执行循环， 从队列中取出任务不为null方法返回 或者
            // poll出异常了重试
            // 2.timeOut == true && timed ==
            // false:看后面的代码workerQueue.poll超时时timeOut才为true，
            // 并且timed要为false，这两个条件相悖不可能同时成立（既然有超时那么timed肯定为true）
            // 所以超时不会继续执行而是return null结束线程。
            if (wc <= maximumPoolSize && !(timedOut && timed))
                break;
            // workerCount递减，结束当前thread
            if (compareAndDecrementWorkerCount(c))
                return null;
            c = ctl.get();         
            // 需要重新检查线程池状态，因为上述操作过程中线程池可能被SHUTDOWN
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
        try {
            // 1.以指定的超时时间从队列中取任务
            // 2.core thread没有超时
            Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();
            if (r != null)
                return r;
            timedOut = true;        // 超时
        } catch (InterruptedException retry) {
            timedOut = false;       // 线程被中断重试
        }
    }
}
```
