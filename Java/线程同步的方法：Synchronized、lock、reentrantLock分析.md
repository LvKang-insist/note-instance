 Synchronized：
 
####  Synchronized修饰的代码块或者方法被某个线程获取到之后，其他线程就会被阻塞。当被修饰的方法执行完后则自动释放锁


Lock：

#### Lock是一个接口，lock提供比Synchronized更广泛的锁操作，他们允许更灵活的结构化可能具有完全不同的属性 ，并且可以支持多个相关联的对象Condition 。

下面列出Lock的常用方法：
- void lock():

获得锁

- void lockInterruptibly():

获取锁，如果可用则立即返回，如果不可用，则处于休眠。

- Condition newCondition();

返回Condition绑定该实例的Lock实例。

- Boolean tryLock():

如果可用，则获取锁，并返回true。如果锁不可用，则返回false。

- Boolean tryLock(long time, TimeUnit unit)：

如果获取到锁，则立即返回true，否则会等待给定的参数时间，在等待的过程中，获取到返回true，等待超时返回false。

- void unlock():

释放锁


注意：当一个线程获取了锁之后，是不会被interrupt()方法中断的，interrupt方法只能中断被阻塞的线程。如果要中断，只能在加锁的方法里面手动实现。


区别：
- 当使用lockInterruptibly()方法的时候，如果没有获取到锁，则线程进入休眠，但是在休眠的过程中，当前线程可以被其他线程中断（会受到InterruptedException异常），也就是锁有两个线程 A,B,获取锁，B获取到锁了，然后A进行休眠，但是这个时候可以调用A.interrupt()方法中断A的休眠。
- 当使用synchronized修饰的时候，当一个线程处于等待的时候是不能被中断的。


---

ReentrantLock：

##### 是可重入锁，ReentrantLock是唯一实现了Lock接口的类，并且提供了更多的使用方法。


ReentrantLock的使用:

请观察如下代码：

```

public class Demo5 {

	public static void main(String[] args) {
		Demo5 demo = new Demo5();
		
		new Thread(new Runnable() {
			
			@Override
			public void run() {
				demo.Test();
			}
		}).start();
		
		new Thread(new Runnable() {
			
			@Override
			public void run() {
				demo.Test();
			}
		}).start();
		
	}
	
	public void Test(){
		Lock lock = new ReentrantLock();
		
		lock.lock();//获取锁
		
		try{
			System.out.println(Thread.currentThread().getName()+"获得了锁");
			Thread.sleep(2000);
		}catch(Exception e){
			e.printStackTrace();
		}finally {	
	    	lock.lock();
			System.out.println(Thread.currentThread().getName()+"释放了锁");
		}
	}
}
```
结果如下：

```
Thread-0获得了锁
Thread-1获得了锁
Thread-0释放了锁
Thread-1释放了锁

```
为什么呢？因为lock是局部变量，每次执行的时候都会创建一个他的对象，所以获取锁的时候获取的就是不同的锁了。
修改的时候只需要将	Lock lock = new ReentrantLock(); 设置为成员变量即可。 

---

tryLock方法的使用：

```
public class Demo5 {
	Lock lock = new ReentrantLock();
	public static void main(String[] args) {
		Demo5 demo = new Demo5();
		
		new Thread(new Runnable() {
			
			@Override
			public void run() {
				demo.Test();
			}
		}).start();
		
		new Thread(new Runnable() {
			
			@Override
			public void run() {
				demo.Test();
			}
		}).start();
		
	}
	
	public void Test(){
		
		
		if(lock.tryLock()){
		
			lock.lock();//获取锁
			
			try{
				System.out.println(Thread.currentThread().getName()+"获得了锁");
				Thread.sleep(2000);
			}catch(Exception e){
				e.printStackTrace();
			}finally {	
				System.out.println(Thread.currentThread().getName()+"释放了锁");
				lock.unlock();
			}
		}else{
			System.out.println(Thread.currentThread().getName()+"获取锁失败");
		}
	}
}

```
结果如下：

```
Thread-0获得了锁
Thread-1获取锁失败
Thread-0释放了锁
```
如上，tryLock是用判断当前锁的状态，获取到返回true，否则返回false。

---
中断线程

```
public class Demo5 {
	Lock lock = new ReentrantLock();
	public static void main(String[] args) {
		Demo5 demo = new Demo5();
		
		Thread t1 = new Thread(new Runnable() {
			
			@Override
			public void run() {
				try {
					demo.Test();
				} catch (InterruptedException e) {
					System.out.println(Thread.currentThread().getName()+"线程被中断");
					e.printStackTrace();
				}
			}
		});
		
				
		Thread t2 = new Thread(new Runnable() {
			
			@Override
			public void run() {
				try{
				demo.Test();
				}catch(Exception e){
					System.out.println(Thread.currentThread().getName()+"线程被中断");
				}
			}
		});
		
		t1.start();
		t2.start();
		t2.interrupt();
	}
	
	public void Test() throws InterruptedException{
		
		
			lock.lockInterruptibly();//获取锁
			
			try{
				System.out.println(Thread.currentThread().getName()+"获得了锁");
				Thread.sleep(2000);
			}catch(Exception e){
				e.printStackTrace();
			
			}finally {	
				System.out.println(Thread.currentThread().getName()+"释放了锁");
				lock.unlock();
			}
	}
}

```
结果如下：

```
Thread-0获得了锁
Thread-1线程被中断
Thread-0释放了锁
```
只有调用lockInterruptibly方法获取的锁才可以中断。

---
Lock和Synchronized的区别:
- Lock是java语言写的，是一个接口。而Synchronized是java内置的。
- lock获取的锁是可以中断的，而Synchronized则是一直阻塞，直到获取锁。
- Synchronized执行完后会自动释放锁，而lock必须手动的释放锁，不然容易造成死锁。
- 通过lock可以知道当前线程有没有获取到锁。

---
重入锁

```
public  synchronized void method1(){	
			System.out.println("method1------开始");
			try {
				Thread.sleep(2000);
				method2();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println("method1------结束");
	
	}
	
	public synchronized void method2(){	
		
			System.out.println("method2------开始");
			try {
				Thread.sleep(2000);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			System.out.println("method2------结束");
	
   }
```

如上所示：
当一个线程执行到method1的时候，获得锁，进入方法然后又会调用method2。这个时候就不需要在获取锁了，因为Synchronized是可重入锁。所以不需要再次获取锁。当然lock也是可重入锁。


---
可中断锁：
在java 中，lock锁可以被中断，但是Synchronized不可以被中断。当一个线程被阻塞的时候，可以中断他的阻塞，然后让他去执行别的任务，这样就是可中断锁。
在上面已经做过演示了。

---
读写锁：ReadWriteLock 他也是一个接口

分为读锁和写锁，多个读锁不互斥，读锁与写锁互斥，这是由jvm自己控制的，你只要上好相应的锁即可。总之，读的时候上读锁，写的时候上写锁！

比如说在写文件的时候，分成2个锁来分配给线程，从而可以做到读和读互不影响，读和写互斥，写和写互斥，提高读写文件的效率。该接口也有一个实现类ReentrantReadWriteLock，下面我们就来学习下这个类。


读操作：
```
public class Demo5 {
	ReadWriteLock lock = new ReentrantReadWriteLock();
	public static void main(String[] args) {
		Demo5 demo = new Demo5();
		
		Thread t1 = new Thread(new Runnable() {
			
			@Override
			public void run() {
				demo.Test();
			}
		});
		
				
		Thread t2 = new Thread(new Runnable() {
			
			@Override
			public void run() {
				demo.Test();	
			}
		});
		
		t1.start();
		t2.start();
	
	}
	
	public void Test() {

			lock.readLock().lock();;//获取锁
			
			try{
				for (int i = 0; i <5; i++) {
					Thread.sleep(100);
					System.out.println(Thread.currentThread().getName()+"读操作");
				}
			}catch(Exception e){
				e.printStackTrace();
			
			}finally {	
				lock.readLock().unlock();
				System.out.println(Thread.currentThread().getName()+"释放了锁");
			}
	}
}

```
结果如下所示：

```
Thread-0读操作
Thread-1读操作
Thread-0读操作
Thread-1读操作
Thread-1读操作
Thread-0读操作
Thread-1读操作
Thread-0读操作
Thread-0读操作
Thread-1读操作
Thread-0释放了锁
Thread-1释放了锁

```
可以看到使用了读锁之后，多个线程可以同时访问Test方法。

写操作

```
public class Demo5 {
	ReadWriteLock lock = new ReentrantReadWriteLock();
	public static void main(String[] args) {
		Demo5 demo = new Demo5();
		
		Thread t1 = new Thread(new Runnable() {
			
			@Override
			public void run() {
				demo.Test();
			}
		});
		
				
		Thread t2 = new Thread(new Runnable() {
			
			@Override
			public void run() {
				demo.Test();	
			}
		});
		
		t1.start();
		t2.start();
	
	}
	
	public void Test() {
		
		
			lock.writeLock().lock();;//获取锁
			
			try{
				for (int i = 0; i <5; i++) {
					Thread.sleep(100);
					System.out.println(Thread.currentThread().getName()+"写操作");
				}
			}catch(Exception e){
				e.printStackTrace();
			
			}finally {	
				lock.writeLock().unlock();
				System.out.println(Thread.currentThread().getName()+"释放了锁");
			}
	}
}
```
结果如下：

```
Thread-0写操作
Thread-0写操作
Thread-0写操作
Thread-0写操作
Thread-0写操作
Thread-0释放了锁
Thread-1写操作
Thread-1写操作
Thread-1写操作
Thread-1写操作
Thread-1写操作
Thread-1释放了锁

```
由结果可得 当使用写锁的时候，只能有一个线程执行Test方法。



---
读写锁 将一个资源分成了两个锁，分别是读锁，和些锁。
可以通过readLock()获取读锁，writeLock()获的写锁。
使用读锁可以被大量的线程访问，但是要保证是同一对象。使用写锁在同一时间内只能被一个线程访问。


---
> 如有错误还请指出，谢谢！