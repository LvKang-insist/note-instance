### 前言

看 WMS 代码的时候看到了 Handler.runWithScissors 方法，所以来恶补一下

```java
public static WindowManagerService main(final Context context, final InputManagerService im,final boolean showBootMsgs, final boolean onlyCore, WindowManagerPolicy policy,ActivityTaskManagerService atm, Supplier<SurfaceControl.Transaction> transactionFactory,Supplier<Surface> surfaceFactory,
            Function<SurfaceSession, SurfaceControl.Builder> surfaceControlFactory) {
        DisplayThread.getHandler().runWithScissors(() ->
                sInstance = new WindowManagerService(context, im, showBootMsgs, onlyCore, policy,
                        atm, transactionFactory, surfaceFactory, surfaceControlFactory), 0);
        return sInstance;
}
```

通过 DisplayThread.getHandler() 调用了 runWithScissors 方法。

该方法的设计初衷就是：**在一个线程中通过 Handler 向另外一个线程发送消息，并等待另一个线程处理完成后再继续执行。**

### runWithScissors

首先来看一下官方文档的描述：

同步运行指定的任务。如何当前线程和处理线程相同，则立即执行不用排队，否则就发送到别的线程进行处理，并等待他完成后再返回。另外，这种方法很危险，使用不当可能会造成死锁，毕竟是两个线程间的通信。

还有该方法被标记为  @hide，因为有一些隐患，所以该方法不希望被开发者使用，一般都用于 Framwork 层。

下面我们来分析一下代码：

```java
public final boolean runWithScissors(@NonNull Runnable r, long timeout) {
    if (r == null) {
        throw new IllegalArgumentException("runnable must not be null");
    }
    if (timeout < 0) {
        throw new IllegalArgumentException("timeout must be non-negative");
    }

    if (Looper.myLooper() == mLooper) {
        r.run();
        return true;
    }

    BlockingRunnable br = new BlockingRunnable(r);
    return br.postAndWait(this, timeout);
}
```

首先获取当前线程的 looper，在拿到 Handler 所属的looper，如果是同一个，就直接执行并返回 true，否则就继续往下走。

如果所属的 looper 不相同，则使用 `BlockingRunnable` 进行包装，并调用 postAndWait 方法：

```java
private static final class BlockingRunnable implements Runnable {
    private final Runnable mTask;
    private boolean mDone;

    public BlockingRunnable(Runnable task) {
        mTask = task;
    }

    @Override
    public void run() {
        try {
            mTask.run();//运行在 handler 线程
        } finally {
            synchronized (this) {
                mDone = true; //标记完成
                notifyAll(); //唤醒线程
            }
        }
    }

    public boolean postAndWait(Handler handler, long timeout) {
        //使用 post 进行发送
        if (!handler.post(this)) {
            return false;
        }

        synchronized (this) {
            if (timeout > 0) {
                final long expirationTime = SystemClock.uptimeMillis() + timeout;
                while (!mDone) {
                    long delay = expirationTime - SystemClock.uptimeMillis();
                    if (delay <= 0) {
                        return false; // timeout
                    }
                    try {
                        wait(delay);
                    } catch (InterruptedException ex) {
                    }
                }
            } else {
                while (!mDone) {
                    try {
                        wait();
                    } catch (InterruptedException ex) {
                    }
                }
            }
        }
        return true;
    }
}
```

在 postAndWait 方法中，首先调用 post 添加到 queue 队列中，如果成功返回 true，如果发送失败，postAndWait 方法直接退出。

发送成功后，就会添加的队里中，等到合适的时候 run 方法就会执行，然后就会执行 finally 块，将 mDone 置为 true。

post() 方法执行成功后，就会进入 `synchronized` 代码块，需要注意的是 run 方法中也有一个 synchronized，这两个锁对象都是 this，所以说，同一时刻只能有一个代码块被执行，另一个只能进行等待。

接着就是 timeout 大于 0 并且 mDone 标志一直处于 false，则进行 wait 等待，等待结束后如果任务还没有完成，直接 return false，表示任务失败。

如果 timeout 小于0，则不需要延时，直接进行阻塞，没有超时时间，只能等待被唤醒。

最后 return true 表示任务成功。

### 梳理流程

1，首先判断目标线程和当前线程是否相同，相同则立即执行任务，return true。

2，接着就使用 BlockingRunnable 进行包装，然后使用 post 发送。发送失败表示目标线程的 Looper 有问题，直接 return false， 表示任务失败。

3，发送成功以后，会有两个分支，一个是 run 方法中的 synchronized，还有一个是 postAndWait 中的synchronized 。这两个在同一时刻只能有一个执行。run 方法中执行任务，postAndWait 中进行延时或者直接等待。

4，最后就是延时等待结束后任务没完成则表示任务失败，如果没有延时就直接进行 wait 进行阻塞，直到被唤醒。这里没有超时逻辑，会存在一定的问题。

### 存在的问题

通过上面的分析，我们大底可以分析出问题的关键了，具体如下所示：

1. 没有超时取消逻辑

    延时完成后，任务如果没有完成，直接回 return false，但是 Runable 依然在运行在目标线程的 MessageQueue 中，最终依然会得到执行，但是不会符合我们的预期

2. 死锁

    1，如果 Runable 在没有执行的时候被移除了，例如 Handler.removeCallBack，Looper.quit，这个任务就永远得不到执行，就会导致 wait 一直等待。

    2，如果 wait 一直无法被唤醒， 并且这个时候还持有者别的锁，就会导致死锁。

那么要如何解决呢，上面第一种也无需解决，如果它不符合你的业务，你也就不需要使用它了，第二种只需要保证当前线程没有别的锁，而且 looper 不能直接退出，需要退出的时候也需要安全退出(quitSafely方法)。

### 总结

通过分析我们也可以看出来 runWithScissors 方法基本上不是偏向于业务的，而是偏向于 framwork 层的，因此该方法被标注为了 hide 方法。如果我们业务真的需要使用这个方法，我们也完全可以仿照源码自己写一个出来，并且还可以随意修改，岂不美滋滋。

好了，本文到这里就结束了，如果对您有用请用您发财的小手点个赞，如果有任何问题可在下方评论。



