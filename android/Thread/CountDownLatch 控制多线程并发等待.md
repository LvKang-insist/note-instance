CountDownLatch 的作用是 倒数，每当一个线程执行完任务后，他就会减一，直到为 0 时 就代表所有的线程的任务都执行完了。总得来说 CountDownLatch 就是等待其他线程执行完任务。

CountDownLatch 主要有两个方法， countDown 和 await 方法，countDown 用于让计数器减一，一般是任务去调用它，await 方法则是使调用该方法的线程处于 等待状态，一般是主线程调用。

看例子：

 当前有一个非常重要的会议，需要所有的高层领导去参加，高层领导共十人，他们代表了十个子线程。但是由于路上耗费了时间，我们需要 等人到齐后才能开始会议。

下面 我们使用 CountDownLatch 来解决这个问题：

1，参与会议的人

```java
public class Participant implements  Runnable{
    private String name;
    private Test test;

    public Participant(String name, Test test) {
        this.name = name;
        this.test = test;
    }

    @Override
    public void run() {
        long duration = (long) (Math.random()*10);
        try {
            TimeUnit.SECONDS.sleep(duration);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        test.save(name);
    }
}

```

2,会议在主线程执行，

```
public class Test {
    CountDownLatch countDownLatch = new CountDownLatch(10);
    public void save(String name) {
        System.out.println(name + " 到了");
        countDownLatch.countDown();
        System.out.println("还剩 " + countDownLatch.getCount() + "  人没到");
    }
    public void start() {
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    public static void main(String[] args) {
        Test test = new Test();
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(new Participant("thread---  " + i, test));
        }
        for (int i = 0; i < 10; i++) {
            threads[i].start();
        }

        test.start();
        System.out.println("人到齐了，开会");
    }
}

```

在最开始我们 创建了一个 Test ，代表着一个会议，接着创建了 10 个子线程，去test开会，只要调用了 save 方法，就说明这个人已经到了。最后调用会议的 start 方法，等待参加会议的人到来。只有人到齐了，主线程才会继续往下走。

结果如下所示：

```
thread---  6 到了
thread---  4 到了
还剩 8  人没到
还剩 9  人没到
thread---  3 到了
thread---  2 到了
还剩 6  人没到
thread---  5 到了
还剩 7  人没到
还剩 5  人没到
thread---  0 到了
还剩 4  人没到
thread---  8 到了
还剩 3  人没到
thread---  1 到了
还剩 2  人没到
thread---  9 到了
还剩 1  人没到
thread---  7 到了
还剩 0  人没到
人到齐了，开会
```

注意CountDownLatch 并不是用阿里保护 资源同步访问的，而是用来控制线程等待的。所以上面打印的有些错乱。但是 主线程一直在等待，当所有人全部到了之后 他才继续往下执行。

一旦CountDownLatch 内计数器为 0，那么调用 countDown 方法不在起作用。

 

