AsyncTask是一个轻量级的异步任务类，它可以在线程池中执行后台任务，然后把执行的进度和结果传递给主线程并且在主线程中更新UI。

AsyncTask 的异步任务是通过 execute 来启动的，我们就以这个为入口，来分析一下

### 1，execute 和 executeOnExecutor 方法

```java
 //Executor是一个接口，SerialExecutor是一个内部类，实现了Executor接口。
 //这是一个向上转型。
 public static final Executor SERIAL_EXECUTOR = new SerialExecutor
 //将SERIAL_EXECUTOR传给sDefaultExecutor。
 private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;


public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}

public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    //判断当前状态
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
    //切换到运行状态
    mStatus = Status.RUNNING;

    //在异步任务之前调用，一般用于初始化，此方法需要被子类重写才可以生效。
    onPreExecute();

    //将传入的参数 给内部抽象类WorkerRunnable的成员数组。
    mWorker.mParams = params;
    
    //回调接口，回调的是SerialExecutor类中所实现的接口。这个会在下面讲到。
    exec.execute(mFuture);

    return this;
}
```
mStatus 代表了当前异步任务的运行状态，我们可以看出AsyncTask 是一次性的，不能重复调用execute 来执行异步任务，当第一次调用时，切换到运行状态，状态为 RUNNING ,接着调用了 onPreExecute()方法，这个方法就是 在异步任务之前调用，需要被重写。接着将我们传入的参数给了  mWorker，

从上面的 sDefaultExecutor 和两个方法可以看出 第二个方法的 exec 参数就是 sDefaultExecutor ，接着我们看一下execute方法。

```java
   
	private final FutureTask<Result> mFuture;
	private static class SerialExecutor implements Executor{
        //双向队列
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;
	   //该方法使用了同步锁。说明添加任务的时候是串行执行的
        public synchronized void execute(final Runnable r) {
        //实例化一个Runnable接口实现run方法，在方法中调用mFuture的run方法。
        //将Runnable的对象添加进队列、即向队列中添加一个新任务
        //这里的Runnable的run方法会在 将队列中的Runnable对象取出来后执行。
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    //r调用的就是mFuture的run方法，
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        //如果当前没有在执行任务，则调用scheduleNext()方法执行下一个任务。
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        //取出mTasks队列中的任务，判断是否为空
        if ((mActive = mTasks.poll()) != null) {
            //6，将取出来的任务进行异步执行，exectute会回调到线程池中执行
            //执行的时候 就会回调mActive的run方法，上面的run方法就会得到执行。
            //在run方法里面就会执行r.run()，执行完后就会再次从队列中去出任务（也就是Runnable对象），
            //直到队列中没有任务
             THREAD_POOL_EXECUTOR.execute(mActive);
        }
    }
}
```
从上面的第一行代码可以看出 这是一个 FutrueTask 的对象，在调用 exec.execute(mFuture) 方法时 将 mFuture 传了进来，接着就是在一个队列中添加了一个Runnable 对象，并在这个 Runnable 的 run 方法中 调用了 mFuture 的run方法，所以只要 队列的中 任务的run 方法执行，那么 mFuture 的run 方法也会执行。

接着就是 mActivie == null 的话就执行 scheduleNext 方法，这个方法会将 队列中的任务拿出来，任务如果不为空，则会交给线程池去执行，当任务被执行 的run 方法被执行，那么上面的 scheduleNext() 方法就会被执行，这样就形成了一个循环，只由队列中没有任务时，这个循环才会停止。

### 2，接着看一下他的构造函数


```java

    private final WorkerRunnable<Params, Result> mWorker;
    private final FutureTask<Result> mFuture;
    
    public AsyncTask() {
        this((Looper) null);
    }
   
    public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);

        //1，创建一个workerRunnable 对象，该类实现了callable接口。
        mWorker = new WorkerRunnable<Params, Result>() {
            //2，实现call()接口，从何处回调在后面会说到，
            public Result call() throws Exception {
              	//在异步任务开启是 添加一个标记
                mTaskInvoked.set(true);
                //AsyncTask的第三个参数，创建一个引用
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    //执行异步任务，将结果返回给result
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    //在异步任务出现异常时 添加一个标记
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    //将结果发送到主线程
                    postResult(result);
                }
                return result;
            }
        };
        //3,创建FutureTask对象，将mWorker的实例对象传进去
        mFuture = new FutureTask<Result>(mWorker) {
            //实现 done方法，在 整个任务完成 或者任务取消时 会执行，默认是空实现
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
    
    //内部抽象类，实现了Callable接口，还有一个Params类型的数组，Params是AsyncTask的第一个参数。
    private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
    }
    
    //FutureTask类的构造函数，该类实现了Rannable，上面(3)处将mWorker传进去(向上转型)，并给成员变量callable。
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
            this.callable = callable;
            this.state = NEW;       // ensure visibility of callable
    }
```
构造方法

首先看一下 mWorker ：

- WorkerRunnable 这个类实现了 Callable 接口，Callable 就是以可以带返回值的方式来创建线程，只不过他需要一个Future 或者 FutureTask 的包装。

- 接着就创建了一个WorkerRunnable的对象，并实现了call方法。这个call 方法 就相当于 Runnable 的run 方法，只不过这个call方法是可以带返回值的。

  可以看到在 call 方法中调用了 doInBackground 方法 （该参数就是 1 中的 mWorker.mParams = params; 这句话），将 doInBackground 的返回值给 result ,最后通过 postResult(result) 将 结果发送到了主线程
  
  ```java
   private Result    postResult(Result result) {
          @SuppressWarnings("unchecked")
          Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                  new AsyncTaskResult<Result>(this, result));
          message.sendToTarget();
          return result;
      }
      
      @SuppressWarnings({"RawUseOfParameterizedType"})
      private static class AsyncTaskResult<Data> {
          final AsyncTask mTask;
          final Data[] mData;
  
          AsyncTaskResult(AsyncTask task, Data... data) {
              mTask = task;
              mData = data;
          }
      }   
  ```
  
  ​	可以看到 ，消息被装进了 内部类 AsyncTaskResult 中，并将该类当称一个消息发送到了主线程。

接下来看一下 mFuture 

- 创建了一个FutureTask对象 mFuture ，并在构造器将WorkerRunnable对象传入，然后实现了done()方法

  ​	接着调用了 postResultIfNotInvoked(get()) 这个方法，并在在参数中 调用了 get 方法，get 方法是一个阻塞方法，他会或者当前线程 执行完之后的返回值，说白了 就是 call 方法的返回值，如果 call 没有执行完，他就一直阻塞。

  ```java
   private void postResultIfNotInvoked(Result result) {
          final boolean wasTaskInvoked = mTaskInvoked.get();
          if (!wasTaskInvoked) {
              postResult(result);
          }
   }
  ```

  ​    首先执行 mTaskInvoked.get() ，还记得 mTaskInvoked 吗，在 call 方法中 设置了 true。他是AtomicBoolean 类的对象，可以保证原子性。如果 为false，则说明 call 方法没有被执行，然后执行 postResult 方法。

### 3，将 1 和 2 连接一下。

​	在 2 中，执行的是构造方法，创建了 FutureTask 对象 mFuture，并传入了一个 Callable 的实例。

​	在 1 中，最后调用了  exec.execute(mFuture) 方法，将mFuture 传进去了。在 exec.execute() 的方法中，将mFuture 对象添加进队列，并依次执行 队列中的任务。 

​	当 队列中的任务执行后，就会执行 mFuture 的run 方法 (队列中的任务就是 Runnable 对象，并且实现了run 方法，在run 方法中调用的是 mFuture 的run方法)。

​	mFuture 的 run 方法是 FutureTask 的run 方法，下面我们看一下 run 方法

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
                    result = c.call();
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

​	上面 第6 行的 callable 就是在 new FutureTask 对象时 传入的 callable 。接着在 11 行执行了 call 方法(AsyncTask 构造方法中的 mWorker) 就会得到执行，接着将结果给 result 。最后调用了 set 方法。

​	在 call 方法中 执行了异步的任务，在执行完后将 结果发送到了主线程。

4，onPostExecute(Result result) 

​		我们都知道 异步完了之后就会执行 onPostExecute(Result result) 这个方法，在call 方法执行完后 将执行的结果发送到了主线程，那么 onPostExecute 方法是从哪里调用的呢？

​	首先看一下构造方法

```java
	private final Handler mHandler;
	public AsyncTask() {
        this((Looper) null);
    }
     public AsyncTask(@Nullable Handler handler) {
        this(handler != null ? handler.getLooper() : null);
    }
    public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
         ......   
    }
    
```

在new AsyncTask 的时候，如果没有传入参数，就会调用 getMainHandler() 方法，

```java
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

看过上面的代码 就清楚了吧，如果没有传入参数，就会自己创建一个 Handler 的实例，用来接收消息。

在 call 方法 执行完后，将结果 装进AsyncTaskResult 类中，然后发送到主线程。在 上面的  handleMessage 方法中，拿到 消息，如果 标记 等于MESSAGE_POST_RESULT ，就调用 AsyncTask 的finish 方法，

```java
 	private void finish(Result result) {
        if (isCancelled()) {
            onCancelled(result);
        } else {
            onPostExecute(result);
        }
        mStatus = Status.FINISHED;
    }

    public final boolean isCancelled() {
        return mCancelled.get();
    }

```

​	isCancelled() 如果为真，则说明在 执行异步的时候发生了异常 或者 当前任务被 取消，然后调用 onCancelled 方法，否则 就调用  onPostExecute(result) 方法，这个方法就是我们需要重写的方法，异步完成后就会调用这个方法。

​	最后将 状态 改为完成。这样整个任务就执行完成了。我将这个过程总结了一下，画了一张图：

​	![1561533950975](F:\笔记\android\Thread\assets\1561533950975.png)

以上就是 AsyncTask 的执行过程了

### 内部线程池的实现

```java
    // 核心线程数量
    private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
    //线程池的最大线程数
    private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
    // 非核心线程 闲置时超时时长，超过这个时长 核心线程就会被回收
    private static final int KEEP_ALIVE_SECONDS = 30;
	
	//线程工厂，为线程池提供创建新线程的能力
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
	//队列，通过线程池的 execute 方法提交的 Runnable 对象会存储在这个参数中。
    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        //创建线程池
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        //闲置核心线程在等待新任务时 会有超时策略，时间由构造方法第三个参数指定。
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```



