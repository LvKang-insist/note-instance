### 抽象类与接口的区别

抽象类和接口都是为了抽取共性而存在的

- 抽象类无法被实例化，只能被继承，抽象类是用来创建继承层级里子类的模板，抽象类可以对方法提供默认的实现
- 抽象类除过不能被实例化和抽象方法以外，其他的和普通类都差不多
- 接口也无法被实例化，如果一个类实现了一个接口，那么它就必须实现所有的方法，这就像是契约模式
- 一个类可以实现多个接口，但是只能继承一个抽象类
- 接口中的变量全部是 final 不可修改，即接口本身只能定义常量。接口是更高级更纯粹的抽象类，体现了对修改关闭，对扩展开放
- 接口在 JDK8 之后可以有不用实现的方法



### final，static，和 synchronized 的作用

#### final 

final 代表了不可变，final  经常配合 static 使用，作为常量

- 用在在类上表示该类不能被继承，
- 在方法上表示不能被重写
- 用在属性上必须赋初始值，往后不允许修改。如果值是引用类型的，那么他不能引用别的对象，但是对象中的属性是可以改变的

final 方法比普通的方法要快，因为在编译的时候已经静态绑定了，不需要运行时动态绑定

#### Static

static 代表着静态，可修饰内部类，方法，字段。

- 修饰内部类：被修饰的类可以直接当做普通的类来使用，不需要先创建一个外部类的实例
- 修饰方法：调用方法的时候不需要依赖任何对象，只需要使用 类名.方法 就可以直接调用，不需要创建对象。
- 修饰变量：通过 类名.变量 的方式就可以直接赋值。静态变量在内存中只有一个副本，在类初次加载的时候会被初始化

#### synchronized

同步代码块，可以用在方法上，或者直接作为代码块使用

- 修饰方法：在同一时刻只能有一个线程访问这个方法，其他线程都会被阻塞
- 修饰代码块：和修饰方法差不多，只不过修饰代码块可以使用类锁。方法如果要使用类锁，就只能修改为静态方法



### String、StringBuffer、StringBuilder 的区别

- String ：常量，一旦创建则不能被修改，是线程安全的，并且 String 被 final 修饰，不可被继承
- StringBuffer：字符串变量，长度是可变的，是线程安全的，适合多线程下的字符串拼接操作
- StringBuilder：字符串变量，长度可变，线程不安全，适合单线程下的字符操作

字符串操作执行速度：StringBuilder > StringBuffer > String



### equals 与 == 、hashCode 的区别和使用场景

一般情况下 equals比较的是 对象的地址，和 == 一样，但是有很多类重写了equals 方法，比如String 等。

 hashcode( ) 方法返回一个 int 数，在object 类中的默认实现是 “将该对象的内存地址 转换成一个整数返回”

```
在java 程序执行期间，在同一对象上调用多次 hashCode 返回的 int 数 必须相同。前提是对象上的equals 方法中所用的信息没有被修改。

如果 equals 比较的两个对象是相等的。那么hashCode 返回的 int 数必定是相等的

当equals 方法被重写时 ，通常有必要重写 hashCode 方法，以维护 hashCode 方法的常规协定。
```

1，若重写了 equals 方法，则有必要重写hashCode 方法

2，若 equals 比较的结果相等，那么 hashCOde 的结果一定相等。

3，若equals 比较的结果不相等，那么 hashCode 的结果 也有可能相等

4，若 hashCode 返回相同的 int 数，那么equals 不一定相等。

5，若 hashCode 返回不同的 int 数，那么 equals 一定不相等。

6，同一对象 在执行期将 若已经存储在集合中。则不能修改影响hashCode 值得相关信息。否则会导致 内存泄露的问题。



### java 深拷贝与浅拷贝的区别

被拷贝的类需要实现 `Clonenable` 接口，该接口为标记接口，不包含任何方法

被拷贝的类重写 clone() 方法，在其中实现具体的拷贝逻辑



浅拷贝：创建一个新对象，将原有对象的属性值进行拷贝，如果是基本类型，则直接拷贝值，如果是引用类型，则直接拷贝对应的引用地址。如果其中一个对象引用地址中的属性发生变化，就会影响另一个对象

深拷贝：拷贝对象的所有属性，如果是引用类型，则会将其引用的对象也一起拷贝，如此循环下去。深拷贝相比与浅拷贝开销较大且速度慢。



### Error 和 Exception 的区别

Exception 是程序运行中可以预料的情况，这种异常可以通过 try catch 进行捕获。

Error 是不可预料的异常情况，这种情况发生后，会直接导致 JVM 不可处理或者不可恢复。这种异常不能被捕获，例如 OutOfMemoryError ，NoClassFoundError 等。



### 什么是反射机制，应用场景有哪些

Java 的反射机制是在运行的过程中，对于任何一个类我们都可以知道他的所有属性和方法，并且可以调用任意属性方法，这种可以动态的调用对象的功能称为 Java 语言的反射机制

应用场景：例如 动态代理，Retrofit就是通过动态代理接口来实现的调用接口方法完成网络请求的，还有 gson，android 中将 xml 解析为 类也是用的反射。

示例：

```java
  //第一种
	Class<?> c1 =new Person().getClass();
	//第二种
	Class<?> c2 = Person.class;
	//第三种，要加包名
	try {
		Class<?> c3 = Class.forName("one.Person");
	} catch (ClassNotFoundException e) {
		// TODO Auto-generated catch block
		e.printStackTrace();
	}
```

### Java 的 char 是两个字节，如何存 UTF-8 字符

- java 中的 char 是用两个字节来存的。

- UTF-8 可能是 1-3个字节

对于这个问题，首先要知道 char 是什么，UTF-8 是什么

- char 里面存的是 Unicode 的码点，Unicode 是字符集，如 ASCII 码也是字符集

  字符集完成字符到整数的映射：字符=》 码点

- UTF-8 是编码，字符集不是编码

  UTF-8 和 UTF-16 的区别是，前者最小的单位是一个字节，后者最小的单位是2个字节

- Java char 不存 UTF-8 的字节，而是 UTF-16 的字节
- Unicode 通用的字符集占两个字节，例如 "中"
- Unicode 扩展字符集需要用一对 char 来表示，例如 "enoji"
- Unicode 是字符集，不是编码，作用类似于 ASCII 码
- Java String 的 length 不是字符数，而是 char 的个数



### Java String 可以有多长

- Java String 字面量形式

  在代码中直接创建的字符串是保存在栈中的，例如 `String str = "aa..aaaa"`。

  编译为字节码之后的表现形式就是 CONSTANT_Utf8_info 了，这是一种结构，如下：

  ```
  CONSTANT_Utf8_info{
  	u1 tag;
  	u2 length;
  	u1 bytes[length]
  }
  ```

  上面 u2 表示的两个字节的一个值，两个字节的值中最大的就是 65535 了。也就是说 length 最大师 65535，bytes 中最多也就可以存 65535 个字节了。

  所以 str 的最大的长度必须小于 65535。为啥要小于 65535 呢，因为编译器在判断长度的时候是判断必须 < 65535，否则的话就会报错。所以如果你定义的字符串长度刚好是 65535 ，那么就会报错。主要注意的是这是 java 编译器的一个bug，如果你用的是 kotlin 那么就不会有这个问题。

  如果字符串保存的是汉子，就需要注意有些汉子是两个字节，而有些是三个字节

  最终 string 是保存在 Java 虚拟机中方法区的常量池中

  总结：

  - 首字节码限制，字符串最终的字节数不能超过 65535 个
  - Latin 字符，受 Javac 代码限制，最多 6534 个
  - 非 Latin 字符最终对应的字节个数差异焦点，最多字节个数是 65535 个
  - 如果运行时方法区比较小，也会受到方法区大小的限制。如果超出了就会内存溢出

- Java String 运行时创建在堆上的形式

  从文件中读出来的字节数组，然后 `new String(bytes)`。这种情况下是存在堆内存中的。

  String 的背后实际上是一个 char 的数组，这个数组受到虚拟机指令的限制，理论最大个数为 Integer.MAX_VALUE ，

  总结：

  - 受虚拟机指令限制，字符数理论上限为 Integer.MAX_VALUE ，
  - 受虚拟机实现限制，实际上可能会小于 Integer.MAX_VALUE。
  - 如果堆内存较小，也会受到堆内存的限制

### Java 匿名内部类的限制

首先，匿名内部类并不是没有名字，他的名字是 "包名+外部类的类名+$N"  , 其中，N 是内名内部类的顺序。例如`com.example.demo.Test$1`。

- 匿名内部类继承结构：

  匿名内部类在 new 出来的时候，就直接是一个子类了。还有就是在 Java 中匿名内部类是不能实现接口的，如下：

  ```java
  new View(context).setOnClickListener(new View.OnClickListener() {
      @Override
      public void onClick(View v) {}
  });
  //错误
  new View(context).setOnClickListener(new View.OnClickListener() implements Runnable{
      @Override
      public void onClick(View v) { }
  });
  ```

  上面的匿名内部类本身就是 `OnClickListener` 的子类了。如果在实现一个 runnable 就直接报错了，这种方式是错误的。但是可以通过方法内部类来实现这种方式：

  ```java
  private void test(Context context) {
      class Listener extends Test implements View.OnClickListener {
          @Override
          public void onClick(View v) {
  
          }
      }
      new View(context).setOnClickListener(new Listener());
  } 
  ```

  如上所示就可以了。

  需要注意的是在 kotlin 中匿名内部类是可以实现接口的，如下：

  ```kotlin
  View(requireContext()).setOnClickListener(object : Test(), View.OnClickListener{
      override fun onClick(v: View?) {
      }
  })
  ```

- 匿名内部类的构造方法

  匿名内部类的构造方法是由编译器为其生成的，构造方法的第一个参数就是当前内部类对应外部类的引用了。如果匿名内部类的父类是非静态类，也需要把非静态的外部类传进来。这样的话就可以在内部类中使用外部的成员和父类的成员了。例如：

  

  如果匿名内部类实现的是接口，那么构造方法就只有一个外部类的引用了。此时就相当于是一个静态类了。

