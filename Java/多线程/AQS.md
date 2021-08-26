### AQS 

`AQS` 全称 `AbstactQueuedSynchronizer` ，即抽象的队列同步器，是一个实现了FIFO 等待队列的同步器，**大白话说就是 AQS 中有一个 FIFO 队列 ，如果要操作共享资源，就会判断当前没有用工作线程在操作这个资源，如果有就会将当前线程加入到队列中，如果没有没有线程操作这个资源，就会将请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。

#### state

AQS 使用了 int 类型的变量 state 来表示同步的状态

- stata = 0 ：表示已经释放了锁
- state > 0 ：表示已经获取到了锁

AQS 提供了三个方法用来修改/获取同步的状态 

- getState()：获取当前同步状态
- setState()：设置新的状态
- compareAndSetState()：以 CAS 的方式来修改状态

#### AQS 相关方法

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210819184130.png" alt="image-20210819184130436" style="zoom: 50%;" />

上述方法可分为三类，

1. 独占式的获取和释放
2. 共享式的获取和释放
3. 同步状态查询和同步队列中的线程等待情况

可重写的方法

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210819184924.png" alt="image-20210819184924340" style="zoom:50%;" />

#### CHL 队列锁

CHL 名字的由来是用三个人名字的首字母连起来的，所以被称之为 CHL 队列

CHL 是一个基于链表的可扩展，高性能，公平的自旋锁，申请资源的线程仅仅在本地变量上自旋，不断轮训判断前节点的状态，发现结点点释放了锁，就会结束自旋，获取锁。

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210820151308.png" alt="image-20210820151307962" style="zoom:50%;" />

完整流程如下：

1. 首先创建一个节点，如上图(Node B) ，队列中已存在排队的节点，就会将其插入到队列的尾部，并使得 myPre 指向前一个节点，设置 locked 为 false。这个时候由于没有获取到锁，则当前线程就会进行自旋进行重试。

   

   如果 `tryAcquire` 没有获取到锁，就会 调用 `addWaiter` 方法 创建节点将其添加到队列尾部，并调用 `qcquireQueued` 方法

2. 后续的线程如果需要使用资源，就需要重复上面的动作

3. 当某个线程要释放锁的时候，就会将当前结点的 locked 置位 false。因为后面的节点都在自旋，当当判断到前节点的 locked 等于 false，那么自身就可以获取到锁，并且释放对前节点的引用，方便 GC 回收

#### 从 ReentrantLock 原理来看 AQS 的独占式锁

**独占式和共享式的最主要区别在于同一时刻独占式只有一个线程获取同步状态，而共享式在同一时刻可以有多个线程获取同步状态。**

ReentrantLock 是基于 AQS 实现的可重入锁，因为 `Synchronized` 关键是用于加锁，但是这种锁对性能影响比较大，因为在线程在获取资源的时候是处于等待状态的，所以说在 JDK 1.5 的时候，Java 提供了 `RenntrantLock `，用于替代  `Synchronized`。

- 使用方式

  ```kotlin
  val reentrantLock = ReentrantLock()
  fun fun1() {
      //获取锁
      reentrantLock.lock()
      //....
  
      //释放锁
      reentrantLock.unlock()
  }
  ```

- ReentrantLock 和 AQS 的关系

  <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210820172329.png" alt="image-20210820172329861" style="zoom: 50%;" />

  ReentrantLock 内部类 Syn 实现了 AQS ，并且派生出两个子类，分别是公平锁和非公平锁

- 流程剖析

  - lock()

    ```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    ```

    调用  `tryAcquire` 尝试获取锁，获取失败则插入到尾结点进行等待

    ```java
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        //没有加锁的线程
        if (c == 0) {
            //如果没有等待的则 CAS 修改同步状态
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                //状态同步成功，修改当前拿到锁的线程
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //如果锁已经被当前线程所持有，则重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            //重新设置状态，
            setState(nextc);
            return true;
        }
        return false;
    }
    ```

    上面代码中会判断在队列没有等待节点的时候尝试修改同步状态，修改成功则拿到锁，否则就会获取当前拿到锁的线程，去判断是否是重入，如果是则就会修改同步的状态。

    ```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //获取上一个节点
                final Node p = node.predecessor();
                //上一个节点如果是头结点就重新尝试获取锁
                if (p == head && tryAcquire(arg)) {
                    //获取成功之后将当前节点设置为头结点
                    setHead(node);
                    //将上一个节点的 next 置位null，方便 gc 回收
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //获取锁失败时调用
                //判断当前节点的线程是否应该被挂起，如果应该则进行挂起
                //等待 release 唤醒释放
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            //在队列中取消当前节点
            if (failed)
                cancelAcquire(node);
        }
    }
    ```

    **总结**：当我们调用 lock 方法时，内部会先当前锁的状态 state：

    1. 状态 == 0 ，没有被持有，并且队列中没有进行等待的，则以原子的方式尝试获取锁，获取成功之后还需要判断是否为锁重入

    2. 获取锁失败，则会创建一个节点，并添加到队列尾部。接着就会不断地进行自旋判断上一个节点是不是 head，如果是就会重新尝试获取，如果不是就会被挂起，等待唤醒

       当然这个自旋的操作不是一直进行下去的，AQS 对这个做了很多处理，当经历一段自旋后，他就会被挂起，等待被唤醒

    其实上面的代码只是一个大概的流程，具体的实现我们不必太过深究，只需要知道他的实现原理即可。

  - unLock

    ```java
    public final boolean release(int arg) {
        //释放锁
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                //传入头结点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    ```

    ```java
    protected final boolean tryRelease(int releases) {
        //计算状态
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        //如果等于 0 表示不是重入锁，则修改持有锁的线程为 null，并更新 状态
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
    ```

    上面代码中主要的就是判断这个锁是否为可重入的，如果是则只更新状态，不是重入锁则需要唤醒其他线程

    ```java
    private void unparkSuccessor(Node node) {
        //...
        
        //查找需要唤醒的结点
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            //唤醒结点
            LockSupport.unpark(s.thread);
    }
    ```

    **总结**：可以看到释放锁还是比较简单的，如果是重入锁则不需要释放锁，只需要修改状态即可，如果不是重入锁则就会遍历找到需要唤醒的节点，进行唤醒

- 非公平锁

  上面的流程剖析中所有的代码都是公平锁(FairSync) 的，最后我们看一下非公平锁的实现

  ```java
  final boolean nonfairTryAcquire(int acquires) {
      final Thread current = Thread.currentThread();
      int c = getState();
      if (c == 0) {
          if (compareAndSetState(0, acquires)) {
              setExclusiveOwnerThread(current);
              return true;
          }
      }
      else if (current == getExclusiveOwnerThread()) {
          int nextc = c + acquires;
          if (nextc < 0) // overflow
              throw new Error("Maximum lock count exceeded");
          setState(nextc);
          return true;
      }
      return false;
  }
  ```

  看着上面代码，如果不是仔细看，你都不会发现他们有啥区别  ![img](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210820185008.png)，仔细看你就会发现，在 状态 = 0 的时候，内部没有判断队列里是否有排队的结点，而是直接尝试获取状态。这就是公平和非公版的区别 （哈哈哈哈哈)

  总结一下：

  - 公平锁：在同步状态之前会判断一下是否有结点在进行等待。如果在等待直接同步失败，然后去排队
  - 非公平锁：在同步状态之前没有判断，直接进行同步，成功就直接修改状态，失败了就乖乖进行排队

#### 共享式锁

其实和 `ReentrantLock` 差不了太多，只是多了个 `ReadLock` 和 `WriteLock` ，分别对应 读锁和写锁。

- 获取共享式锁

  ```java
  public final void acquireShared(int arg) {
      if (tryAcquireShared(arg) < 0)
          doAcquireShared(arg);
  }
  ```

  首先调用上面的 `tryAcquireShared ` 方法尝试获取同步状态，如果获取失败则调用 `doAcquireShared` 自旋方式获取同步状态，自旋式获取同步状态如下：

  ```dart
  private void doAcquireShared(int arg) {
      // 设置当前节点为共享模式，并且添加达到队列中
      final Node node = addWaiter(Node.SHARED);
      boolean failed = true;
      try {
          boolean interrupted = false;
          for (;;) {
              //获取前驱节点
              final Node p = node.predecessor();
              if (p == head) {
                  //尝试获取同步
                  int r = tryAcquireShared(arg);
                  //获取到同步
                  if (r >= 0) {
                      //设置为队列的头
                      setHeadAndPropagate(node, r);
                      //释放前驱节点的 next，方便回收
                      p.next = null; // help GC
                      if (interrupted)
                          //中断线程
                          selfInterrupt();
                      failed = false;
                      return;
                  }
              }
              //获取锁失败时调用
              //判断当前节点的线程是否应该被挂起，如果应该则进行挂起
              //等待 release 唤醒释放
              if (shouldParkAfterFailedAcquire(p, node) &&
                  parkAndCheckInterrupt())
                  interrupted = true;
          }
      } finally {
          if (failed)
              cancelAcquire(node);
      }
  }
  ```

- 释放锁

  ```java
  public final boolean releaseShared(int arg) {
      if (tryReleaseShared(arg)) {
          doReleaseShared();
          return true;
      }
      return false;
  }
  ```

  ```java
  private void doReleaseShared() {
          for (;;) {
              //获取头结点
              Node h = head;
              //头结点不为 null && 不等于尾结点
              if (h != null && h != tail) {
                  //获取等待状态
                  int ws = h.waitStatus;
                  if (ws == Node.SIGNAL) {
                      if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                          continue;            // loop to recheck cases
                      //唤醒后继结点
                      unparkSuccessor(h);
                  }
                  else if (ws == 0 &&
                           !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                      continue;                // loop on failed CAS
              }
              if (h == head)                   // loop if head changed
                  break;
          }
      }
  ```

  这个释放过程和独占的释放过程不太相同，共享式释放锁的过程中，对于能够支持多个线程访问的并发组件，必须要保证能够安全的释放同步锁。上面使用了 CAS ,失败之后再下循环中重试

#### 自己实现一个可重入锁

> 可重入锁，指的就是某个线程已经获取锁之后，调用了某个方法又获取锁，这个时候如果不是可重入，则线程会被阻塞，造成死循环。
>
> 可重入的实现原理其实非常简单，只要在获取锁的时候判断当前线程已经获取到了锁，则将状态 + 1，表示重入成功即可，需要切记的是在释放的时候需要将的状态-1，

```java
class SelfReentrantLock {

    private val sync = Sync()

    fun lock() {
        //内部会调用 tryAcquire 方法
        sync.acquire(1)
    }

    fun unlock() {
        sync.release(1)
    }

    fun tryLock(): Boolean {
        return sync.tryAcquire(1)
    }

    fun tryLock(time: Long, unit: TimeUnit): Boolean {
        return sync.tryAcquireNanos(1, unit.toNanos(time))
    }


    class Sync : AbstractQueuedSynchronizer() {
        override fun isHeldExclusively(): Boolean {
            return state == 1
        }

        //获取锁
        public override fun tryAcquire(arg: Int): Boolean {
            //类似于 CAS 操作，返回 true 表示已经拿到锁
            if (compareAndSetState(0, 1)) {
                //设置当前锁的持有者
                exclusiveOwnerThread = Thread.currentThread()
                return true;
            } else if (exclusiveOwnerThread == Thread.currentThread()) {
                state++
                return true
            }
            //失败表示锁被占用
            return false
        }

        //释放锁
        public override fun tryRelease(arg: Int): Boolean {
            if (state == 0) {
                throw IllegalMonitorStateException()
            }
            state -= 1
            if (state == 0)
                exclusiveOwnerThread = null
            return true
        }
    }

}

val lock = SelfReentrantLock()
fun main() {

    Thread(Runnable {
        sync()
    }).start()

    Thread(Runnable {
        sync()
    }).start()

}

fun sync() {
    lock.lock()
    println("获取锁 —— ${Thread.currentThread().name}")
    Thread.sleep(1000)
    reentrant()

    lock.unlock()
    println("释放锁 —— ${Thread.currentThread().name}")
}

fun reentrant() {
    lock.lock()
    println("获取锁 —— ${Thread.currentThread().name}")


    lock.unlock()
    println("释放锁 —— ${Thread.currentThread().name}")
}
```

### volatile

只能作用于变量，只能保证可见性，并且不能保证原子性，

- 可见性：当修改被 volatile 修饰的变量时，修改之前会强制从主内存从获取一下最新值，修改完之后会立即将修改过的值刷新到主内存中。也就是说，对于一个 volatile 变量的读，总是能看到(任意线程)对 volatile 这个变量的写入

- 原子性：对于单个 volatile 变量的读和写具有原子性，但类似于  volatile a++ 这种复合操作是不具有原子性的

- 抑制重排序

  流水线和重排序

  其实 cpu 不像我们想的那样同时只能执行一条指令，其实他可以同时执行多条指令

  ```kotlin
  fun do(){
  	int a = 4
  	int b = 10
  	int c = 20
  	int d = 12
  	
  	if(a = 4){
  		a=b
  	}
  }
  ```

  一般情况下，执行顺序是 a= 4,然后 b = 10 .... 一直到执行完

  但是引入了流水线和重排序以后，cpu 就可以同时执行多条指令，例如上面代码可能在同一时刻执行了上面的四条代码（如下图所示），在这个同时也执行了 a = b，这个时候就会把 a = b 的结果放在一个叫重排序缓存的空间里面，等到真正执行到 if(a=4) 并且成立的时候，就会直接从 重排序缓存里面将这个值拿出来直接使用，而不用在进行计算。

  <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210819180942.png" alt="image-20210819180942624" style="zoom: 50%;" />

  当然上面这种方式只有在上下代码没有关联的时候才会使用。这种就是现代 CPU 为了提高性能而提出的流水线和重排序的功能。

  这种重排序的功能在单线程下，不管是在 cpu 还是在 jvm 上面，都能够保证最后计算出的结果是符合预期的，

  但是在多线程中就可能会出现重排序导致的顺序混乱，所以 volatile 还有一个关键的作用就是抑制重排序，使用了 volatile 关键字后，cup 或者 jvm 就不会对这个变量进行重排序，从而使得结果保证预期

Volatile 实现原理

​	被 volatile 修改的变量在进行读写操作的时候会是哦用 CPU 提供的 Lock 前缀指令，这个指令是由现代 CPU 所提供的一个特殊的指令，这个指令的主要作用：

1. 将当前处理器缓存行的数据写回到系统内存

2. 这个写回的操作会使得在其他 CPU 里缓存了该内存地址的数据无效

   当某个线程修改了 volatile 变量之后，其他线程里面的该变量会被设定为无效，在使用的时候就会强制重新读取

   
