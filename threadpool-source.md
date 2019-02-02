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
        // 获取当前线程状态
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
