
#####  本文将对Android消息机制的实现原理做一个分析，由于Android的消息机制实际上就是Handler的运行机制，分别是Handler，MessageQueue和Looper。同时也说一下主线程的消息循环。

---

#### 1，主线程的消息循环:                    
   
   
  android的主线程就是ActivityThread，主线程的入口方法为main，在main方法中通过Looper.prepareMainLooper()来创建主线程的Looper以及MessageQueue。并通过loop()方法来开启主线程的消息循环。
    
    
```
public static void main(String[] args) {
        ......

        Process.setArgV0("<pre-initialized>");

        //创建主线程的Looper和MessageQueue.
        Looper.prepareMainLooper();

        // Find the value for {@link #PROC_START_SEQ_IDENT} if provided on the command line.
        // It will be in the format "seq=114"
        long startSeq = 0;
        if (args != null) {
            for (int i = args.length - 1; i >= 0; --i) {
                if (args[i] != null && args[i].startsWith(PROC_START_SEQ_IDENT)) {
                    startSeq = Long.parseLong(
                            args[i].substring(PROC_START_SEQ_IDENT.length()));
                }
            }
        }
        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        //开启消息循环
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

```
Looper类中的相关方法：
```
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }

    //创建主线程的Looper
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```

```
    //Looper类的构造方法
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
    //返回Looper的对象
     public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

```
public static void loop() {
        //贴出相关源码....

        //获取looper对象，myLooper的作用：返回sThreadLocal存储的Looper实例；若me为null 则抛出异常
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        //通过Looper实例中MessageQueue的对象
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        
        boolean slowDeliveryDetected = false;

        for (;;) {
            //从消息队列中取出消息
            Message msg = queue.next(); // might block
            
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            try {
                //msg.target 是handler对象，调用自己的dispathMessage()方法处理消息。
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            ......
            msg.recycleUnchecked();
        }
    }
```

下面我们 分析一下这段代码
- 在main方法中通过调用Looper.prepareMianLooper()来创建Looper对象和MessageQueue对象，观察源码可知在 prepareMainLooper()方法的第一行调用了prepare()方法，在这个方法中，首先会判断Lopper的对象是否为空。这里涉及到一个知识点 ThreadLocal 。这是一个线程内部的数据存储类。[不太懂了小伙伴可以看下这篇博客](https://blog.csdn.net/baidu_40389775/article/details/86759882)。如果Looper的对象为null，他就会创建一个对象 然后通过ThreadLocal将Lopper的对象保存起来（这里保存的是主线程创建的Lopper对象）接下来查看Loopper的构造方法可知，在创建Lopper对象的时候 已经创建好了MessageQueue 对象，也拿到了当前线程的实例。由此也可以看清 在一个线程中 只能有一个Looper对象，否则会抛异常

- 返回到用Looper.prepareMianLooper()方法中.直接看最后一行，通过myLopper()方法直接获取到Looper的对象然后给sMainLooper。

- 最后 在 看一下main方法中的looper.loop()方法，loop方法是一个死循环，唯一跳出循环的方式是MessageQueue的next方法返回了null。在loop方法的第一句 先得到 当前线程的looper对象。如果为空直接抛出异常。往下看 是一个死循环，循环的第一句调用next()从消息队列中取出一条消息，这个方法是阻塞的。没有消息时next方法会一直阻塞在哪里。这也导致loop方法一直阻塞在哪里。当next方法返回了新的消息，Looper就会处理这条消息： msg.target.dispatchMessage(msg) msg.target是发送这条消息的Handler对象，这样handler发送的消息最终又交给它的dispatchMessage方法来处理了。
Looper类 有两个方法来通知消息队列的退出，quit或者quitSafely方法来通知消息队列的退出，当消息队列被标记为退出状态时 ，它的next方法就会返回null。


---
    

### 2,Handler的工作原理

handler的主要作用就是用来发送和接收消息，发送可以通过send和post等方法来实现。post的一系列方法最后还是通过send来实现的。下面我们来读一下源码

##### 首先看一下他的构造方法


```
     public Handler() {
        this(null, false);
    }
    
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        //获取当前线程的Looper对象
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        //获取Looper对象中保存的 消息队列对象
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    
```
从上面 可以看出 要想创建handler对象，就必须先有Looper对象，否则直接抛出异常，所以在子线程中创建handler对象的时候必须先创建Looper对象。也可以通过Looper.getMainLooper() 获取主线程的Looper对象。（主线程会默认创建Looper对象）


##### 然后就是发送消息和处理消息


```
    public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
    
     public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
     public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
    
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
    
```

可以看到当调用sendMessage发送消息时，到最后只是将这条消息放在了消息队列中。在将消息放在队列之前将handle的对象传给了msg.target，这也就是为什么在Looper的loop方法中可以使用msg.target的原因。看了上面主线程的消息循环可知，在Looper的loop方法中 会不断从消息队列中读取消息，当读取到消息的后 Looper会将读取到的消息交给handle的dispatchMessage方法来处理。这时handler就进入到了处理消息的阶段


```
public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
handler通过dispatchMessage方法来处理消息，首先判断callback是否为空，不为空则是post发送的消息，这里在handleCallback方法里面直接回调 执行post发送时实现的run方法。

```
private static void handleCallback(Message message) {
        message.callback.run();
    }
```
如果为空则 说明是send发送的消息 然后判断mCallback是否为空。mCallback是一个接口，先来看一下他的定义:


```
/**
     * Callback interface you can use when instantiating a Handler to avoid
     * having to implement your own subclass of Handler.
     */
    public interface Callback {
        /**
         * @param msg A {@link android.os.Message Message} object
         * @return True if no further handling is desired
         */
        public boolean handleMessage(Message msg);
    }
```
当mCallback 不为空 就说明handler没有子类，不需要实现handleMessage方法，如下所示

```
 Handler handler = new Handler(new Handler.Callback() {
            @Override
            public boolean handleMessage(Message msg) {
                return false;
            }
        });
```
在创建 handler对象的时候 实现接口，就可以拿到从子线程发送的消息。为什么要使用这个呢，源码里面是这样说的：在实例化处理程序时可以使用回调接口来避免
必须实现自己的处理程序子类。 在日常使用handler的时候，一般都是派生一个handler的子类 重写他的handleMessage方法。但是Callback给我们提供了另一种使用方式，当我们不想派生子类的时候就可以使用这种方式。

### 3，消息队列的工作原理
消息队列指的是 MessageQueue ，MessageQueue主要包含两个操作，插入和读取，对应的方法分别是 enqueueMessage 和 next 。enqueueMessage是插入一条消息，next 从消息队列中读取一条消息并将其移除。虽然 MessageQueue 叫消息队列，但是 它里面实现的是单链表。下面主要看一下他的插入和读取的方法。

```
boolean enqueueMessage(Message msg, long when) {
       ......
        synchronized (this) {
            ......
            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            //判断 链表中有没有消息和when 是否满足条件，
            //if成立，新消息则作为头结点，
            //if不成立，新消息插入到 链表中合适的位置
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
由上可知，这里的主要操作就是单链表，满足条件插入到最前面，否则插入到合适的位置。


```
Message next() {
        //贴出关键代码......
        
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg ;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                
                //出队消息，把消息从链表中取出，按时间顺序
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                     // 若 消息队列中已无消息，则将nextPollTimeoutMillis参数设为-1
                // 下次循环时，消息队列则处于等待状态
                    nextPollTimeoutMillis = -1;
                }
              .....
            }
        ......
        }
    }
```
可以发现 next 是一个无限循环的方法，如果没有消息next就会一直阻塞到这里，有消息时 next方法会将这条消息返回并移除此消息。

---

### 4，Looper的工作原理
looper具体的作用就是不断的从消息队列中取出消息，有消息就进行处理，没有消息就一直阻塞。观察源码可知（源码在 上面讲主线程消息循环的时候已经贴出来 并且说了一下运行过程）这里简单的说一下就好

- 构造方法，Looper的构造方法里面会创建一个MessageQueue的对象，并且将当前线程保存起来。
- 创建Looper，如果Handler 使用的时候 是在子线程创建的，那么就需要手动的开启一个Loopoer。当然主线程的Looper在源码中已经创建好了，所以在主线程中不需要自己开启Looper。通过调用Looper.prepare()方法即可为当前线程创建一个Looper，接着通过loop方法来开启消息循环，这样loop就会不断的从消息队列中获取消息。
- prepareMainLooper() 这个方法是给主线程创建Looper使用的，本质也是通过prepare来实现的，Looper还提供了一个getMainLooper()可以拿到当前looper的对象。
- Looper的退出，Looper提供了quit和quitSafely来退出Looper，两者的区别是quit会直接退出Looper，而quitSafely只是设定一个标记，当队列中消息处理完后 才会退出Looper。Looper退出后 handler发送的消息会失败，send方法就会返回false。在子线程种如果创建了Looper，那么在使用完成后一定要quit终止消息，否则这个线程 就会一直处于阻塞状态。
- loop()方法，这个方法在讲主线程消息循环的时候 已经说过了，不太懂的可以在看一下。
 

---

### 5，Message：

Message是在多个线程之间传递的消息，其内部可以携带少量的信息，用于在不同的线程中进行交互。


---
基本的已经说完了，下面看一下使用的过程。

1. 当程序开始运行的时候，在ActivityThread的main方法中（也就是主线程中），创建了Looper对象，Looper的构造器中也创建了消息队列的对象，通过ThreadLocal将Looper进行保存，最后调用loop方法开启消息循环。
2. 创建handler的对象，在handler的构造器中通过myLooper拿到了主线程的Looper，然后通过looper拿到了消息队列的对象。
3. 在子线程中使用handler.sendMessage()发送消息，最后会调用enqueueMessage()将消息添加到消息队列中。注意在发送消息的时候 最后会将当前handler的实例传入到消息中。
4. Looper的loop方法检测到消息队列中有消息，然后获取一个可以处理的消息。从获取的消息中拿到handler的实例然后调用dispatchMessage()方法。然后消息又回到了handler中。
5. 最后在handler的dispatchMessage()方法中对消息进行处理。然后根据不同的情况分别进行处理。这样就是Handler机制的整个流程。


---

> 如有错误，还请指出，谢谢。
