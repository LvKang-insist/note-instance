#### 序列化是将对象保存在磁盘中，或允许在网络中直接传输对象。对象序列化机制允许把内存中的java对象转成与平台无关的二进制流，从而将这种二进制文件持久的保存在磁盘上。其他程序一旦获得了这种二进制的流。就可以将这个二进制流恢复成原来的java对象


---

对象的序列化：指将一个java对象写入到IO流中，与此对应的是，对象的反序列化则是从IO流中恢复该对象.


如果要将某个对象保存到磁盘上或者通过网络传输，那么这个类应该实现Serializable接口或者Externalizable接口之一。

使用Serializable来实现序列化非常简单，主要让目标类实现Serializable标记接口即可，无需实现任何方法。

---

一旦某个类实现了Serializable接口，该类的对象就是可以序列化的.


```
package cn.lvkang.com.gradletest;

import java.io.BufferedWriter;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.OutputStreamWriter;
import java.io.Serializable;

public class SerTest implements Serializable {

    private int age = 25;
    private String name ="张三";

    private SerTest() {
        System.out.println(this.age+"-----------"+this.name);
    }

    public static void main(String[] args) {
        
        //对对象进行序列号
        try {
            ObjectOutputStream fos = new ObjectOutputStream(new FileOutputStream("object.txt"));
            SerTest ser = new SerTest();
            fos.writeObject(ser);

        } catch (Exception e) {
            e.printStackTrace();
        }

        //从文件中读取该对象，成为反序列化
        try {
            ObjectInputStream ois = new ObjectInputStream(new FileInputStream("object.txt"));
             SerTest serTest = (SerTest) ois.readObject();
            System.out.println(serTest.age);
            System.out.println(serTest.name);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}

```
必须指出的是反序列化 的仅仅是java对象的数据，而不是java类，因此采用的反序列化恢复java对象时，必须提供该Java对象所属类的class文件，否则会引发ClassNotFoundException异常

还有一点，SerTest只有一个构造器，而且构造器中只有一个打印语句。而在反序列化时，并没有看到程序调用该构造器，这表明反序列化机制无需通过构造器来初始化java对象。

---


#### 对象引用的序列化

如果某个类的成员变量不是基本类型或者String类型，而是一个引用类型，那么这个引用的类必须是可序列化的，否则拥有该引用变量的类也是不可序列化的.

如下：

```
class Person implements Serializable {
    String name;
    SerTest test;

    public Person(String name, SerTest test) {
        this.name = name;
        this.test = test;
    }
}
```
Person 持有SerTest的引用，只有SerTest是可序列化的，Person才可以被序列化，


假设有如下情景：

```
        SerTest test = new SerTest();
        Person p1 = new Person("张三",test);
        Person p2 = new Person("李四",test);
```
这里产生了一个问题，在序列化p1的时候，系统会将p1对象所引用的test对象一起序列化，如果程序在序列化p2的时候，系统一样会序列p2，并且序列化test。从而引起p1和p2使用的不是同一个对象，显然这个就违背了java序列化的初衷。所以java采用了一种特殊的算法。算法内容如下：
- 所有保存在磁盘中的对象都有一个序列化的编号。
- 当程序视图序列化一个对象时，程序将先简称该对象是否被序列化过，只有该对象从未被序列化过，系统才会将对象转换成字节序列并输出
- 如果这个对象已经被序列化过，程序将只是输出一个序列化编号，而不是重新序列化该对象。

根据上面的算法，可以知道 在程序在序列化p2的时候，发现test已经被序列化过了，所以程序不会对test进行序列化，3而是输出一个序列化编号。


---


#### 自定义序列化：
在一些特殊的情况下，如果一个类中包含某种特殊的信息，如银行账户信息时，这是不希望将该实例变量值进行序列化，或者这个类的某个变量是不可被序列化的，因此不希望对该实例遍历进行递归序列化。

当某个对象进行序列化时，系统会自动把该对象的所有实例变量依次进行序列化，如果这个类的实例变量引用到其他类的对象，则被引用的对象也会被序列化，如果被引用的对象的实例变量也引用了其他类，在被引用的对象也会被实例化，这种情况称之为递归序列化。

通过在实例变量前面使用 transient关键字修饰， 可以指定java序列化时无需理会该实例变量.如下所示

```
public class SerTest implements Serializable {

    private transient int age = 25;
    private String name ="张三";

    private SerTest() {
        System.out.println(this.age+"-----------"+this.name);
    }

    public static void main(String[] args) {

        //对对象进行序列号
       try {
            ObjectOutputStream fos = new ObjectOutputStream(new FileOutputStream("object.txt"));
            SerTest ser = new SerTest();
            fos.writeObject(ser);

        } catch (Exception e) {
            e.printStackTrace();
        }

        //从文件中读取该对象，成为反序列化
        try {
            ObjectInputStream ois = new ObjectInputStream(new FileInputStream("object.txt"));
             SerTest serTest = (SerTest) ois.readObject();
            System.out.println(serTest.age);
            System.out.println(serTest.name);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```
添加了transient 之后，该类的age属性将不会被序列化,同样的在进行反序列化的时候age是没有值的也就是为0。


#### 对敏感的字段进行加密:

在序列化的过程中，虚拟机是试图调用对象里的writeObject和readOjbect方法，如果没有这样的方法，则默认调用的是ObjectOutputStream的defaultWriteObject 方法以及 ObjectInputStream的 defaultReadObject方法。用户自定义的writeObject和readObject方法可以允许用户控制序列化的过程。比如可以在序列化的过程中动态改变序列化的数组，基于这个原理，可以在实际应用中使用，可以对敏感的字段进行加密的工作。


```
public class SerTest implements Serializable {

    private static ObjectOutputStream fos;
    private static ObjectInputStream ois;
    //敏感字段 年龄和 姓名
    private  int age = 28;
    private String name = "张三";

    private SerTest() {

    }

    //对象在序列化的时候调用
    private void writeObject(ObjectOutputStream fos) throws IOException {
        System.out.println("--------------加密中--------------");
        System.out.println("原来的姓名："+this.name);
        System.out.println("原来的年龄："+this.age);
        StringBuffer buffer = new StringBuffer(this.name);
        //对姓名进行反转
        fos.writeObject(buffer.reverse());
        //对年龄进行加密
        fos.writeInt(this.age*10*(5-2));
    }


    //对象在反序列化的时候调用
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        System.out.println("--------------加密后--------------");
        StringBuffer buffer = (StringBuffer) ois.readObject();
        this.name = buffer.toString();
        System.out.println("加密后的姓名："+this.name);
        this.age = ois.readInt();
        System.out.println("加密后的年龄："+this.age);
    }

    public static void main(String[] args) {
        try {
            SerTest test = new SerTest();
            fos = new ObjectOutputStream(new FileOutputStream("obj.obj"));
            fos.writeObject(test);//进行序列化
            fos.close();

            
            ois = new ObjectInputStream(new FileInputStream("obj.obj"));
            //进行反序列化，并且进行敏感字段的解密
            SerTest t = (SerTest) ois.readObject();

            System.out.println("---------------解密中-------------------");
            StringBuffer buffer = new StringBuffer(t.name);
            System.out.println("解密后的姓名："+buffer.reverse().toString());
            System.out.println("解密后的年龄："+(t.age/10/(5-2)));
            ois.close();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

    }

}
```

在程序中使用了自定义序列化，在序列化的时候ObjectOutputStream /ObjectInputStream 会根据你传入的对象进行反射。判断你是否写了writeObject/readObject 方法。如果写了，就会调用你写的，否则就会调用默认的方法.

==在序列化的时候对对象里面的数据进行了加密，然后取出来的时候又进行了解密。这就是对象的自定义序列化==

---


### 静态变量序列化：

序列化保存的是对象的状态，而不是类的状态。静态变量属于类的状态，所以序列化的时候不会保存静态常量。



---


#### writeReplace 方法：

还有一种更彻底的自定义机制，他甚至可以自序列化的时候将该对象转为其他的对象。writeReplace将由序列化机制调用，只有该方法存在，就会被调用。如下所示：

```
public class SerTest implements Serializable {

    private static ObjectOutputStream fos;
    private static ObjectInputStream ois;
    //敏感字段 年龄和 姓名
    private  int age = 28;
    private String name = "张三";

    private SerTest() {
    }


    private Object writeReplace() throws ObjectStreamException {
        System.out.println("序列化中..............");
        ArrayList<Object> list = new ArrayList<>();
        list.add(new Person("李四"));
        return list;
    }



    public static void main(String[] args) {
        try {
            SerTest test = new SerTest();
            fos = new ObjectOutputStream(new FileOutputStream("obj.obj"));
            fos.writeObject(test);//进行序列化
            fos.close();


            ois = new ObjectInputStream(new FileInputStream("obj.obj"));

            ArrayList t = (ArrayList) ois.readObject();
            System.out.println(t.get(0).toString());
            ois.close();
            
            
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

    }

}

class Person implements Serializable {
    String name;
    public Person(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return name;
    }
}
```
打印结果如下

```
序列化中..............
李四
```

==系统在序列化某个对象之前，会先调用该对象的WriteReplace方法。如果该方法返回的是另一个对象，系统则会调用另一个对象的writeReplace方法---直到该方法不在返回另一个对象为止。程序最后将调用该对象的writeObject()方法来保存该对象的状态。==


由打印的结果可以看出，在序列化的时候看似序列化的SerTest，实际上序列化的是ArrayList。在ArrayList中添加了一个Person对象，该对象也实现了序列化接口。但是并没有实现writeReplace方法。所以最后将会调用writeObject保存该对象的状态。

注意：上面集合里面存的是person对象，并没有writeReplace方法，所以最后将调用该对象的writeObject()方法来保存该对象的状态。但是如果保存的是当前类的对象(SerTest类的对象),就会==造成递归，然后程序就会直接挂了。== 因为SerTest对象有writeReplace方法。程序会一直的调用这个方法。最后程序会直接挂掉。如下所示：

修改writeReplace方法

```
 private Object writeReplace() throws ObjectStreamException {
        System.out.println("序列化中..............");
        ArrayList<Object> list = new ArrayList<>();
//        list.add(new Person("李四"));
        list.add(new SerTest());
        return list;
    }
```
打印结果如下：

```
......
序列化中..............
序列化中..............
Exception in thread "main" java.lang.StackOverflowError
	at sun.nio.cs.UTF_8.updatePositions(UTF_8.java:77)
	at sun.nio.cs.UTF_8.access$200(UTF_8.java:57)
	at sun.nio.cs.UTF_8$Encoder.encodeArrayLoop(UTF_8.java:636)
	at sun.nio.cs.UTF_8$Encoder.encodeLoop(UTF_8.java:691)
	at java.nio.charset.CharsetEncoder.encode(CharsetEncoder.java:579)
	at sun.nio.cs.StreamEncoder.implWrite(StreamEncoder.java:271)
	at sun.nio.cs.StreamEncoder.write(StreamEncoder.java:125)
	......

```

直接报 堆栈异常。

---

#### ReadResolve()方法：
该方法和writeReplace方法对应。他可以实现保护性的复制整个对象，该方法会紧跟着readObject()之后调用，该方法的返回值将会代替原来反序列化的对象，而原来readObject()反序列化的对象将会被立即丢弃。


---
Externalizable 序列化机制，这种序列化方式完全由程序员决定存储和恢复对象的数据。要想使用Externalizable，必须实现这个接口。

该接口定义了两个方法。
- writeExternal():

需要序列化的类实现writeExternal()方法来保存对象的状态.该方法调用的是DataOutput(他是ObjectOutput的父接口)的方法来保存基本类型的实例变量的值，调用ObjectOutput的writeObject()方法来保存引用类型的实例变量值。
- readExternal：

需要序列化的类实现readExternal方法来实现反序列化。该方法调用DataInput(他是ObjectInput的父接口)的方法来恢复基本类型的实例变量的值，调用ObjectInput的readObject()方法来恢复引用类型的实例变量值。

实际上，采用实现Externalizable接口的方式 和前面说的自定义序列化十分像是，只是这个强制实现了自定义序列化。如下所示：

```
public class ExterTest implements Externalizable {
    private static ObjectOutputStream out;
    private static ObjectInputStream in;
    private String name ="王五";
    private int age = 35;

    public ExterTest(){
        System.out.println("我是构造器");
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeObject(new StringBuffer(this.name).reverse());
        out.writeInt(this.age);
    }

    @Override
    public void readExternal(ObjectInput in) throws ClassNotFoundException, IOException {
        this.name = ((StringBuffer)in.readObject()).reverse().toString();
        this.age = in.readInt();
    }

    public static void main(String[] args){
        ExterTest test = new ExterTest();

        try {
            out = new ObjectOutputStream(new FileOutputStream("test.obj"));
            out.writeObject(test);
            out.close();


            in = new ObjectInputStream(new FileInputStream("test.obj"));
            ExterTest t = (ExterTest) in.readObject();
            System.out.println(t.name+"-------"+t.age);

            in.close();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

```
上面程序实现了Externalizable接口，也实现了两个方法，这两个方法除了名字和readOjbect()/writeObject()不同外，其他方法体都一样.

如果程序需要序列化显示Externalizable接口的对象，一样调用OabjectOutputStream的writeObject()方法即可。

需要说明的是：当使用这个方式反序列化时，程序首先会使用public的无参构造器创建实例。然后在执行readExternal()方法进行反序列化，因此实现Externalizable的接口必须提供public的无参构造器.


#### 关于对象的序列化，还有一下几点需要注意：
- 对象的类名，实力变量(包括基本数据类型，数组，对其他对象的引用)都会被序列化。方法、类变量、transient实例变量都不会被序列化。
- 实现Serializable，接口的类如果需要让某个实例变量不被序列化，这个在该实例变量前加transient修饰符，而不是static关键字。
- 保证序列化对象的实例变量类型也是可以被序列化的，否则需要使用transient关键字来修饰该实例变量。不然该类是不可被序列化的。
- 反序列化的时候北徐有序列化对象的class文件。
- 当通过文件、网络来读取序列化后的对象时，必须按实际写入的顺序读取.


#### 版本：

根据前面所说的，反序列化是必须有该对象的class文件，现在问题是随着项目的升级，系统的class文件也会跟着升级，java如果保证两个class的兼容性？

- java序列化机制允许为序列化的类提供一个private static final 的serialVersionUID值，该类的变量用于表示java类的序列化版本。也就是说，当一个类升级后，只要该类变量的值没有修改，序列化机制也会把他们当作为同一个序列化版本。

- 最好在每个要序列化的类中加入private static final long serialVersionUID这个类变量的值，值自己定义。这样该类被修改了，该对象也能被序列化.


如果不显示的定义该值，该类变量的值将会有jvm进行计算，而修改后的类的值往往和没修改的值不同，从而容易造成对象反序列化的时候因为版本问题而导致的无法被序列化。

