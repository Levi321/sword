[toc]

# AsyncTask 的使用

Android 提供 AsyncTask 处理异步任务，基于异步消息处理机制，本质上是一个封装了线程池和Handler的异步框架定义如下：

```java
public abstract class AsyncTask<Params, Progress, Result> {
    ...
}
```

## 三个泛型参数

1. Params：执行 Asynctask 时需要传入的参数，可用于后台任务中使用
2. Progress：执行时的进度
3. Result：执行完毕后返回的结果

## 重写四个方法

1. onPreExecute()：异步任务开启之前回调，在主线程中执行
2. doInBackground()：执行异步任务，在线程池中执行
3. onProgressUpdate()：当 doInBackground() 中调用 publishProgress() 时回调，在主线程中执行
4. onPostExecute()：在异步任务执行之后回调，在主线程中执行

# 源码分析

## Android 3.0 之前的 Asynctask

```java
public abstract class AsyncTask<Params, Progress, Result> {
    private static final int CORE_POOL_SIZE = 5;
    private static final int MAXIMUM_POOL_SIZE = 128;
    private static final int KEEP_ALIVE = 1;
    private static final BlockingQueue<Runnble> sWorkQueue = 
        new LinkedBlockingQueue<Runnble>(10);
    ...
    private static final ThreadPoolExecutor sExecutor = 
        new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE
                              TimeUnit.SECONDS, sWorkQueue, sThreadFactory)
    ...
}
```

这里可以看到 ThreadPoolExecutor 的核心线程数是 5，最大线程数是 128，任务队列的容量是 10，所以 AsyncTask 最多同时能容纳 138（128 + 10）个任务，当提交第 139 个任务时就会执行饱和策略，抛出RejectedExecutionException 异常。

## Android 7.0 的 Asynctask

```java
public abstract class AsyncTask<Params, Progress, Result> {
    private static final int CORE_POOL_SIZE = 1; // 使用单核心线程
    private static final int MAXIMUM_POOL_SIZE = 20;
    private static final int BACKUP_POOL_SIZE = 5;
    private static final int KEEP_ALIVE_SECONDS = 3; // 等待超时时间 3s

    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    // 饱和策略，当任务队列满了，并且达到了最大线程数后使用,懒加载，最好用永远不要初始化
    private static ThreadPoolExecutor sBackupExecutor;
    private static LinkedBlockingQueue<Runnable> sBackupExecutorQueue;
    private static final RejectedExecutionHandler sRunOnSerialPolicy =
            new RejectedExecutionHandler() {
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            synchronized (this) {
                if (sBackupExecutor == null) {
                    sBackupExecutorQueue = new LinkedBlockingQueue<Runnable>();// 容量 Integer.MAX_VALUE
                    sBackupExecutor = new ThreadPoolExecutor(
                            BACKUP_POOL_SIZE, BACKUP_POOL_SIZE, KEEP_ALIVE_SECONDS,
                            TimeUnit.SECONDS, sBackupExecutorQueue, sThreadFactory);
                    sBackupExecutor.allowCoreThreadTimeOut(true);
                }
            }
            sBackupExecutor.execute(r);
        }
    };

    // 用于并行处理任务
    public static final Executor THREAD_POOL_EXECUTOR;
    // 初始化 THREAD_POOL_EXECUTOR
    static {
        // 核心数 1，最大线程数 20，等待超时 3s，
        // SynchronousQueue 是一个不存储元素的阻塞队列。每个插入操作必须等待另一个线程的移除操作，同样任何一个移除操作都必须等待另一个线程的插入操作。因此对列内部其实没有一个元素，或者说容量是 0
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(), sThreadFactory);
        threadPoolExecutor.setRejectedExecutionHandler(sRunOnSerialPolicy);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }

    // 用来串行处理任务的 Executor
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static final int MESSAGE_POST_RESULT = 0x1;
    private static final int MESSAGE_POST_PROGRESS = 0x2;

    @UnsupportedAppUsage
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    private static InternalHandler sHandler;

    @UnsupportedAppUsage
    private final WorkerRunnable<Params, Result> mWorker;
    @UnsupportedAppUsage
    private final FutureTask<Result> mFuture;

    @UnsupportedAppUsage
    private volatile Status mStatus = Status.PENDING;
    
    private final AtomicBoolean mCancelled = new AtomicBoolean();
    @UnsupportedAppUsage
    private final AtomicBoolean mTaskInvoked = new AtomicBoolean();

    private final Handler mHandler;
```

关键方法：

```java
@MainThread
protected void onPreExecute() {
}

@WorkerThread
protected abstract Result doInBackground(Params... params);

@MainThread
protected void onProgressUpdate(Progress... values) {
}

@MainThread
protected void onPostExecute(Result result) {
}
```

execute()：

```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```

execute(Params... params) 直接调用了 executeOnExecutor(sDefaultExecutor, params)

```java
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) {
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }
    mStatus = Status.RUNNING;
    onPreExecute();
    mWorker.mParams = params;
    exec.execute(mFuture);
    return this;
}
```

executeOnExecutor(sDefaultExecutor, params) 先进行了状态校验，防止运行多次；然后调用了 onPreExecute()，之后 为mWorker 赋值，先看一下 mWorker 和 mFuture 是什么：

```java
@UnsupportedAppUsage
private final WorkerRunnable<Params, Result> mWorker;

public AsyncTask(@Nullable Looper callbackLooper) {
    ...
    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            mTaskInvoked.set(true);
            Result result = null;
            try {
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                //noinspection unchecked
                result = doInBackground(mParams);
                Binder.flushPendingCommands();
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
                postResult(result);
            }
            return result;
        }
    };
    
    mFuture = new FutureTask<Result>(mWorker) {
        @Override
        protected void done() {
            try {
                postResultIfNotInvoked(get());
            } catch (InterruptedException e) {
                android.util.Log.w(LOG_TAG, e);
            } catch (ExecutionException e) {
                throw new RuntimeException("An error occurred while executing doInBackground()",
                        e.getCause());
            } catch (CancellationException e) {
                postResultIfNotInvoked(null);
            }
        }
    };    
}

private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}
```

mWorker 是一个实现 Callable 接口的 WorkerRunnable，mFuture 即 FutureTask，我们主要看一下这个 FutureTask。

FutureTask 可用于异步获取执行结果或取消执行任务的场景。通过传入 Runnable 或者 Callable 的任务给 FutureTask，直接调用其 run() 或者放入线程池执行，之后可以在外部通过 FutureTask 的 get() 异步获取执行结果，因此，FutureTask 非常适合用于耗时的计算，主线程可以在完成自己的任务后，再去获取结果。

AsyncTask 的 get()：

```java
public final Result get() throws InterruptedException, ExecutionException {
    return mFuture.get();
}
```

这里 AsyncTask 真正干活的就是 mWorker，其中调用了 `result = doInBackground(mParams)`。使用 mFuture 对 mWorker 进行包装，方便外接通过 `mFuture.get()` 获取当前进度。

继续看这个 sDefaultExecutor：

```java
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;

private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;
    public synchronized void execute(final Runnable r) {
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext(); // 
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }
    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```

最核心的地方是这个 sDefaultExecutor，她的 execute(final Runnable r) 传入的 Runnable 对象其实就是 mFuture，`r.run()` 就是 mFuture.run()，FutureTask 的 run() 如下：

```java
public void run() {
    if (state != NEW ||
        !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call(); // aaaaa
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

可以看到 run() 关键的地方是 `result = c.call()`，也就是包装的 Callable.call()，也就是 之前的 mWorker.call()，doInBackground() 之后 调用 postResult(result) 将结果传出：

```java
private Result postResult(Result result) {
    @SuppressWarnings("unchecked")
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
            new AsyncTaskResult<Result>(this, result));
    message.sendToTarget();
    return result;
}
```

postResult 本质上是使用 Handler 发送 result，handler 代码如下：

```java
private Handler getHandler() {
    return mHandler;
}

private final Handler mHandler;

mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
    ? getMainHandler()
    : new Handler(callbackLooper);
    
private static Handler getMainHandler() {
    synchronized (AsyncTask.class) {
        if (sHandler == null) {
            sHandler = new InternalHandler(Looper.getMainLooper());
        }
        return sHandler;
    }
}

private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) {
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```

使用的 handler 是内部封装的 InternalHandler，接收到 MESSAGE_POST_RESULT 消息后，会调用 AsyncTask 的 finish()：

```java
private void finish(Result result) {
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```

finish() 中判断如果任务被取消，走 onCancelled()；如果任务完成了，走 onPostExecute()。

总结整个过程：线程池的 execute() 调用传入的 FutureTask.run() ，mFuture 内包装了真正做工的 mWorker，mWorker本质上是一个 Callable 对象，在 call() 中 调用了 doInBackground() ，并在结束后向 handler 发送完成消息，handler 在接收到消息后，调用 onPostExecute() 结束整个过程。































