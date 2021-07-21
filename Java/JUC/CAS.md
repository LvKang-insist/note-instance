### CAS(乐观锁)

CAS 全称为 `Compare And Swap` 翻译过来就是`比较并且交换`

- Synchornized 是悲观锁，线程一旦得到锁，其他的线程就只能挂起了
- cas 的操作则是乐观锁，他认为自己一定会拿到锁，所以他会一直尝试，直到成功拿到为止；



### CAS 机制

在看到 Compare 和 Swap 后，我们就应该知道，CAS 里面至少包含了两个动作，分别是比较和交换，在现在的 CPU 中，为这两个动作专门提供了一个指令，就是`CAH` 指令，由 CPU 来保证这两个操作一定是原子的，也就是说比较和交换这两个操作只能是`要么全部完成，要么全部没有完成`

CAS 机制中使用了三个操作数，内存地址，旧的预期值，要修改的值；

例如 a + 1 的操作，a 默认=0，

1，在多个线程修改一个值 a 的时候，会将 a copy 一份到自己的线程内存空间中(预期值)，此时预期值就是 a ，要修改的值就是 a+1 的结果，结果就是 1(要修改的值)，由于是多个线程，所以每个线程内部都会得到 a  = 1。

2，接着就会执行**比较并且交换**，  让线程中的预期值和主内存中的 a 进行比较，如果相同，就会提交上去，如果不相同，说明 a 的值已经被别的线程修改过了，所以就会提交失败（这个比较和提交的操作是原子性的）。提交失败之后，线程就会重新获取 a  的值，然后重复这一操作。这种重复操作的方式称为自旋

栗子：

前提：线程 A，B，主内存中的变量 count = 0；

1. 线程A： 要修改 count 值，所以 copy 一份到自己的内存中，然后执行了 + 1 的操作，此时线程A中 count 预期值是 0，要修改的值为 1

2. 线程B ：也修改 count 值，也执行了 + 1 的操作，此时线程 B 中 count 的预期值是 0，要修改的值为 1，

3. 线程B ：开始提交到主内存了，提交的时候判断预期值 和 主内存的 count 是一样的，所以就会提交成功，**这时主内存 count =1**

4. 线程A ：也开始提交了，但是在判断的时候发现预期值是 0，但主内存是1，不相等，所以，提交失败，然后就会放弃本次提交。

5. 线程A ：提交失败之后，就与重新执行步骤 1 的操作。



这种方式最终可以理解成一种 无阻塞的多线程争抢资源的模型。



### 问题

- ABA

  还是上面的栗子，

  在线程 A 执行  + 1，操作的时候，线程 B 已经将 + 1的结果提交的主内存了，但是这个时候他又执行了 - 1的操作提交到主内存，并且这个过程快于线程 A。

  这个时候线程 A 进行判断和交换，发现修改的值和主内存的值相同，然后将计算的结果提交了。

  ___

  在线程 A 执行的过程中，线程 B 修改了值，并且将值又修改了回去，虽然说结果并没有变化，但是值已经被操作过了

  这就是典型的 ABA 问题

  那么如何解决呢？

  ___

  其实解决起来非常简单，只需要增加一个版本戳即可，在更新值的时候判断一下版本戳即可。

  在 Java 中也有使用版本戳的实现，就是 `AtomicMarkableReference` 和 `AtomicStampedReference`。

  `AtomicMarkReference` ：只关心这个变量有没有被动过

  `AtomicStampedReferrence` ：不但关心这个变量有么有动过，并且关心这个变量被动了几次，例如

  ```kotlin
  val asr = AtomicStampedReference<String>("345", 0)
  
  fun main() {
      val oldStamp = asr.stamp
      val oldReference = asr.reference
      println("$oldReference ----  $oldStamp")
  
      val t1 = Thread {
          println("${Thread.currentThread().name}   当前变量值：${asr.reference}  当前版本：${asr.stamp}" +
                  "   ${asr.compareAndSet(asr.reference, "3456", asr.stamp, asr.stamp + 1)}")
      }
      val t2 = Thread {
          println("${Thread.currentThread().name}   当前变量值：${asr.reference}  当前版本：${asr.stamp}" +
                  "   ${asr.compareAndSet(asr.reference, "34567", asr.stamp, asr.stamp + 1)}")
      }
      t1.start()
      t1.join()
      t2.start()
      t2.join()
  
      println("${asr.reference} ----  ${asr.stamp}")
  }
  345 ----  0
  Thread-0   当前变量值：345  当前版本：0   true
  Thread-1   当前变量值：3456  当前版本：1   true
  34567 ----  2
  ```

  在修改字符串的时候，要传入已经修改过的字符串和版本号，负责就会修改错误

- 开销问题

  在 CAS 期间，线程是不会休息的，线程如果长时间无法提交，内部就一直在进行自旋，这样就会产生比较大的内存开销

- CAS 只能够保证一个共享变量的原子操作

  CAS 只能保证对一个内存地址进行原子操作，所以说使用范围会有一定限制

  例如：如果在执行 a+1 的下面加上，b+1，c +1，这种情况就会出现问题，这种时候反而使用 Syn 比较方便
  
  ___
  
  其实 Java 中也提供了可以修改多个变量的原子操作
  
  `AtomicReference`：将需要修改的包装成一个对象，然后使用 `AtomicReference` 的 compareAndSet 方法进行替换即可
  
  ```kotlin
  fun main() {
      val user = User("张三", 31)
      val atomicReference = AtomicReference<User>(user)
      println("${atomicReference.get().name} --- ${atomicReference.get().age}")
  
      atomicReference.compareAndSet(user, User("李四", 20))
      println("${atomicReference.get().name} --- ${atomicReference.get().age}")
  }
  
  class User(val name: String, val age: Int)
  ```
  
  上面的代码就保证了修改多个变量，实际上就是更新对象



### 实例

在 java 的 atomic 包下，一系列以 Atomic 开头的包装类，如 `AtomicBoolean` `AtomicInteger` 等，分别用于 int，bool，long 等类型的原子操作，其原理都是用的 cas

核心实现如下：

```java
//使用将给定函数应用于当前值和给定值的结果，以原子方式更新当前值，并返回更新后的值。 该函数应无副作用，因为当尝试更新由于线程间争用而失败时，可能会重新应用该函数。 应用该函数时，将当前值作为其第一个参数，并将给定的update作为第二个参数 
public final int accumulateAndGet(int x,
                                      IntBinaryOperator accumulatorFunction) {
    //预期值和要更新的值
        int prev, next;
        do {
            //获取到预期值，也就是当前值
            prev = get();
            //计算要更新的值
            next = accumulatorFunction.applyAsInt(prev, x);
            //更新成功则退出循环，否则重新计算
        } while (!compareAndSet(prev, next));
        return next;
}

 //注意：这个比较并且 set 的操作是原子性的
 //参数：期望–期望值  更新–新价值
 //返回值：
 //如果成功，则为true 。 错误返回表示实际值不等于期望值
public final boolean compareAndSet(int expect, int update) {
     return U.compareAndSwapInt(this, VALUE, expect, update);
}
```

