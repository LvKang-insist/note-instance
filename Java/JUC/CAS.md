### CAS(乐观锁)

- Synchornized 是悲观锁，线程一旦得到锁，其他的线程就只能挂起了
- cas 的操作则是乐观锁，他认为自己一定会拿到锁，所以他会一直尝试，知道成功拿到为止；



### CAS 机制

CAS 英文名是 Compare And Swap 缩写，翻译过来就是比较和替换。

CAS 机制中使用了三个操作数，内存地址，旧的预期的值，要修改的值；

在修改一个值A的时候，会将 A copy 一份到自己的线程内存空间中，此时预期值就是 A ，要修改的值就是A+1(也就是线程要如果修改A)，在修改完成之后，线程开始提交更新的值，提交的时候就会让预期值和主内存中的 A 进行比较，如果相同，就会提交上去，如果不相同，说吧 A 的值已经被别的线程修改过了，所以就会提交失败。提交失败之后，线程就会重新获取 A 的值，然后重复这一操作。这种重复操作的方式称为自旋

栗子：

前提：线程 A，B，主内存中的变量 count = 0；

1. 线程A： 要修改 count 值，所以 copy 一份到自己的内存中，然后执行了 + 1 的操作，此时线程A中 count 预期值是 0，要修改的值为 1

2. 线程B ：也修改 count 值，也执行了 + 1 的操作，此时线程 B 中 count 的预期值是 0，要修改的值为 1，

3. 线程B ：开始提交到主内存了，提交的时候判断预期值 和 主内存的 count 是一样的，所以就会提交成功，**这时主内存 count =1**

4. 线程A ：也开始提交了，但是在判断的时候发现预期值是 0，但主内存是1，不相等，所以，提交失败，然后就会放弃本次提交。

5. 线程A ：提交失败之后，就与重新执行步骤 1 的操作。



这种方式最终可以理解成一种 无阻塞的多线程争抢资源的模型。



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

 //如果当前值==期望值，则以原子方式将该值设置为给定的更新值。
 //参数：期望–期望值  更新–新价值
 //返回值：
 //如果成功，则为true 。 错误返回表示实际值不等于期望值
public final boolean compareAndSet(int expect, int update) {
     return U.compareAndSwapInt(this, VALUE, expect, update);
}
```

