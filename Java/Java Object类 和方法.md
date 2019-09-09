public class Object

Object 位于java.long 包中，java.long包是java最核心的类，在编译时会自动导入

Object是 所有类的祖先 每个类都使用Object类作为超类，所有对象（包括数组）都实现了这个类的方法。 


---
#### -  clone():
克隆的意思。必须实现Cloneable接口才可以调用这个方法，主要用途是用来 保存另一个当前存在的对象。
#### - equals(Object obj):
用于比较两个对象是否相同
#### - finalize():
该方法用于释放资源。当垃圾收集确定不再有对该对象的引用是，垃圾收集器在对象上调用该对象。
#### - getClass():
返回当前运行时类。
#### - hashCode():
返回对象的哈希码值。
#### - notify():
唤醒正在等待对象监视器的单个线程。
#### - notifyAll():
唤醒正在等待对象监视器的所有线程。
#### - toString():
返回对象得字符创表现形式。
#### - wait():
导致当前线程等待，直到另一个线程调用该对象的notify()方法或notifyAll()方法。
#### - wait(long timeout):
导致当前线程等待，直到另一个线程调用该对象的notify()方法或notifyAll()方法，或者指定的时间已过。
#### - wait(long timeout,int nanos):
- 导致当前线程等待，直到另一个线程调用该对象的notify()方法或notifyAll()方法，或者某些其他线程中断当前线程，或一定量的实时时间。


---

其我感觉 比较重要的就是hashCode 和 equals()，关于这两个方法的详细用法，[可以看着个博文](https://blog.csdn.net/baidu_40389775/article/details/87173379)