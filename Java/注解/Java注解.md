java 提供了一种源程序中的元素关联 任何信息和任何元数据的途径 和方法。

## Java 中常见的注解

### JDK自带的注解

- @Override

  用来指定方法的重载，他可以强制一个子类必须覆盖父类的方法。

  ```java
  public class Per {
      String name;
  
      public String getName() {
          return name;
      }
  }
  
  class Man extends Per{
  
      //指定下面方法必须重写父类方法
      @Override
      public String getName() {
          return super.getName();
      }
  }
  ```

- @Deprecated

  表示当前(类，方法)等，已经过时，当程序调用这个 类 ，方法时，编译器会给出警告。

  ```java
  public class Per {
      String name;
  
      @Deprecated
      public String getName() {
          return name;
      }
      
      public static void main(String[] args){
          Per per = new Per();
          //调用getName方法时，会给出警告
          per.getName();
      }
  }
  ```

- @Suppvisewarnings

  抑制编辑器警告，被该注解修饰的程序元素 将会被取消显示编译器的警告。

  ```java
  public class Per {
      String name;
  
      @Deprecated
      public String getName() {
          return name;
      }
  
  
      @SuppressWarnings(value = "unchecked")
      public static void main(String[] args){
          Per per = new Per();
          //使用 @SuppressWarnings 后不会再有警告
          per.getName();
      }
  }
  ```

### 注解的分类

- 源码注解：只在源码中存在，编译成 .class 文件后就不存在了。
- 编译时注解：注解在源码和 .class 文件中都存在。上面的三种注解就属于编译时注解，只在编译的时候有效。
- 运行是注解： 在运行阶段还起作用，甚至还会影响运行逻辑。

### 自定义注解

```java
/**
 *  使用 @interface 用来定义注解。
 *
 *  带成员变量的注解
 *      成员变量以无参无异常的方式声明
 *      成员变量以方法的形式来声明
 *      一旦在注解里面定义了成员变量后，使用该注解时 就应该为他的成员变量指定值
 *          (成员如果指定了默认值，则不用)
 */
```

```java
public @interface Description {

    //没有 指定默认值的成员
    String desc();

    //可以使用 default 为成员指定一个默认的值。
    int age() default 20;
}
```

使用如下：

```java
public class Per {
    @Description(desc = "为没有指定默认值的成员 赋值")
    public static void main(String[] args){
        Per per = new Per();
    }
}
```

### 元注解

- @Retention

  该注解只能用于修饰注解定义，用于指定 被修饰的注解可以保留多长时间，包含一个Retention Policy 类型的value成员变量，所以使用该注解时 必须为该注解指定 成员变量的值。

  value 的值只能是如下三个：

  - Retention Policy.CLASS : 编译器 将会把注解记录在 class 文件中，当运行 java 程序时，JVM 不可获取注解的信息。这是默认值。

  - Retention Policy.RUNTIME : 编译器 将会把注解记录在 class 文件中，当运行 java 程序时，JVM 也可以获取注解信息，程序可以通过反射 获取该注解的信息。

  - Retention Policy.SOURCE : 注解只保留在源代码中，编译器直接丢弃这种注解。

    ```java
    // 定义下面的 @Description 注解保留到运行时。
    @Retention(value = RetentionPolicy.RUNTIME)
    public @interface Description {
        String desc();
        int age() default 20;
    }
    ```

    也可以采用入下 代码来为 value 指定值

    ```java
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Description {
        String desc();
        int age() default 20;
    }
    ```

    上面使用 @Retention 元注解时 ，并未通过 value = Retention Policy.RUNTIME 的方式来为该成员指定值，这是因为 当注解的成员变量名为 value 时，程序可以直接在注解后的括号里指定该成员变量的值。

- @Target

  他用于指定被修饰的的注解能用于修饰那些程序单元。@Target 注解也包含一个为 value 的成员变量，该成员变量的值只能是如下几个。

  ![1554977884228](C:\Users\Lv_345\AppData\Local\Temp\1554977884228.png)

  比如第一个，表示 只能修饰 注解，2，只能修饰 构造函数。......

  ```JAVA
  @Target(ElementType.CONSTRUCTOR)
  public @interface Description {
      String desc();
      int age() default 20;
  }
  ```

  上面程序表示 注解只能修饰 构造方法

![1554978173166](C:\Users\Lv_345\AppData\Local\Temp\1554978173166.png)

​	如果用来修饰 方法就会报错。

- @Documented

  指定被该元注解 修饰的注解将被 java doc 工具提取成文档，所有使用该元注解的程序的 API 文档中将会包含该注解的说明

  ![1554987614478](C:\Users\Lv_345\AppData\Local\Temp\1554987614478.png)

  ![1554987644203](C:\Users\Lv_345\AppData\Local\Temp\1554987644203.png)

- @Inherited

  该元注解指定被他修饰的注解将具有继承性，如果某个类使用了 @xxx注解 (定义该注解时使用了 @Inherited)修饰，则其子类将自动被@xxx继承。 

  如下所示：

  ```java
  @Retention(RetentionPolicy.RUNTIME)
  @Inherited
  public @interface Description {
  }
  ```

  上面 注解中使用了 @Inherited 代表具有继承。如果某个类使用了该注解，则他的子类会默认被该注解修饰。注意：@Retention(RetentionPolicy.RUNTIME) 这个必须加。

  ```java
  @Description
  public class Per {
      Per() {}
      public static void main(String[] args){
          new Student();
      }
  }
  
  class Student extends Per{
  
      public Student() {
          super();
          System.out.println(Student.class.isAnnotationPresent(Description.class));
      }
  }
  ```

  打印结果为 true,说明该注解具有继承性。

- @Repeatable

  可重复注解。代表着该注解在某个类或者方法等 上面可以使用多次。要创建可重复注解，必须创建容器注解。然后将该注解的类型作为 @Repeatable 的参数，如下所示：

  ```java
  //使用可重复注解
  @Repeatable(FKTags.class)
  //该注解将被保留在运行时。
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.TYPE)
  @interface FKTag{
  	String name() default "哈哈哈";
  	int age();
  }
  
  //可重复注解的容器，如果上面的注解 在某个类或方法等 上面使用了多次，他的注解就会被保存在该注解的集合中。
  @Retention(RetentionPolicy.RUNTIME)
  @Target(ElementType.TYPE)
  @interface FKTags{
  	FKTag[] value();
  }
  ```

  然后使用注解，提取充 注解的信息

  ```java
  @FKTag(age = 20)
  @FKTag(name = "嘻嘻嘻",age=30)
  public class Tast {
  	
  	public static void main(String[] args) throws ClassNotFoundException {
  		
  		Class<Tast> c = Tast.class;
  		
  		
  		FKTags fk = c.getAnnotation(FKTags.class);
  		for(FKTag f : fk.value()){
  			System.out.println(f.age());
  			System.out.println(f.name());
  		}
  
  		System.out.println("--------------");
          //找出 修饰Tast 类的 @FKTags 注解
  		FKTags f =c.getDeclaredAnnotation(FKTags.class);
  		for(FKTag fktag : fk.value()){
  			System.out.println(fktag.age());
  			System.out.println(fktag.name());
  		}
  	}
  }
  ```

  个人感觉：如果 重复注解使用两次以上，就会将他的这些注解 保存在定义的容器中，然后直接遍历该容器，就可以拿到注解中的信息了。

  如果只是用了一次，就不会放在容器中，遍历容器就会直接报错，这时直接提取该注解信息就好了，而不是 去遍历容器。



### 提取注解的信息

​	使用了注解修饰类、方法、成员变量之后，这些注解不会自动生效，必须由开发者提供相应的工具来提取并处理注解信息。

​	Annotation 定义类型为 @interface ，所有的 Annotation 都会自动继承 java.lang.Annotation 这一接口，并且不能去继承别的类或者接口。

​	要获取类、方法和字段的注解想信息，必须通过 Java 的反射技术来获取 Annotation 对象，除此之外没有别的方法。

​	同时为了运行时能准确的获取注解的相关信息，java 在 java.lang.reflect 反射包下新新增了 AnnotatedElement 接口，态抓以后用于表示目前正在 JVM 中运行的程序中已使用注解的元素。通过该接口提供的方法 可以利用反射技术读取注解的信息，如反射包的Constructor类、Field类、Method 类、Package 类和 Class 类都实现了 AnnotatedElement 接口。

​	![1555054664756](C:\Users\Lv_345\AppData\Local\Temp\1555054664756.png)

​		

​	Class ： 类的Class对象定义

​	Constructor ：代表着构造器的定义。

​	Field ：类的成员变量定义

​	Method ： 类方法的定义

​	Package ：类的包定义

AnnotatedElement 接口 是 所有程序元素(Class , Method, Constructor 等 )的父接口，所以程序通过反射获取了某个类的 AnnotatedElement 对象之后，程序就可调用该对象的 如下几个方法来访问注解的信息。

​	![1555055367546](C:\Users\Lv_345\AppData\Local\Temp\1555055367546.png)

```java
@Retention(RetentionPolicy.RUNTIME)//该注解 生存周期是运行时。
@Target(ElementType.METHOD)//该注解只能用于方法上
public @interface AnnotationTest {
    int value() default 20;
}
```

```java
public class Test {

    private String name;
    private int age;

    @AnnotationTest
    public void setAge(int age) {
        this.age = age;
    }

    @AnnotationTest
    public void setName(String name) {
        this.name = name;
    }

    public static void main(String[] args) throws ClassNotFoundException {
        //加载类 类对象
        Class<?> tclass = Class.forName("cn.lvkang.com.gradletest.annotationtest.Test");
        if (tclass.isAnnotationPresent(AnnotationTest.class)){
            //拿到注解类实例
            AnnotationTest at = tclass.getAnnotation(AnnotationTest.class);
            //因为该注解只用于方法上，所以在这里打印为空。
            System.out.println(at.value());
        }

        //找到使用 该注解的方法
        Method[] methods = tclass.getMethods();
        for (Method m :methods){
           //判断该方法如果使用了注解，则 提取信息
            if (m.isAnnotationPresent(AnnotationTest.class)) {
                AnnotationTest a = m.getAnnotation(AnnotationTest.class);
                System.out.println(a.value());
            }
        }

        //另外一种解析方法
        for (Method m : methods){
            //返回这个方法上面的所有注释
            Annotation[] a = m.getAnnotations();
            for (Annotation annotation: a){
                //判断 该注释 是不是 AnnotationTest的实例
                if (annotation instanceof AnnotationTest){
                    System.out.println(((AnnotationTest) annotation).value());
                }
            }
        }
    }
}
```

### 使用注解

```java
@Retention(RetentionPolicy.RUNTIME)//该注解 生存周期是运行时。
@Target(ElementType.METHOD)//该注解只能用于方法上
public @interface AnnotationTest {
    int value() default 20;
}
```

​	上面的注解用来标记 那些方法是可以进行使用的

```java
//使用 @AnnotationTest 指定该方法是可以测试的。
public class Test {
    @AnnotationTest
    public static void test1(){
    }
    public static void test2(){
    }

    @AnnotationTest
    public static void test3(){
        throw new RuntimeException("参数错误");
    }
    public static void test4(){
    }

    @AnnotationTest
    public static void test5(){
    }
    public static void test6(){
    }

    @AnnotationTest
    public static void test7(){
        throw new RuntimeException("程序业务出现异常");
    }
    public static void test8(){
    }
}
```

上面 加注解的 代表要被执行

```java
public class ProcessorTest {

    public static void process(String clazz) throws ClassNotFoundException {

        int passed  = 0;
        int failed = 0;

        for(Method m : Class.forName(clazz).getMethods()){
            if (m.isAnnotationPresent(AnnotationTest.class)){
                try {
                    //调用m 的方法
                    m.invoke(null);
                    passed++;
                } catch (Exception e) {
                    failed++;
                    System.out.println(m + "方法运行失败，异常"+e.getCause());
                }
            }
        }

        System.out.println("共运行了"+(passed+failed)+"次，成功"+passed+"次，失败"+failed+"次");
    }
}
```

上面的类 的 process 方法会通过 传进去的参数 进行反射，拿到 使用注解的方法，并测试是否可以成功运行。

```java
public class RunTest {
    public static void main(String[] args) throws ClassNotFoundException {
        ProcessorTest.process("cn.lvkang.com.gradletest.annotationtest.Test");
    }
}
```

运行结果如下：

```
public static void cn.lvkang.com.gradletest.annotationtest.Test.test7()方法运行失败，异常java.lang.RuntimeException: 程序业务出现异常
public static void cn.lvkang.com.gradletest.annotationtest.Test.test3()方法运行失败，异常java.lang.RuntimeException: 参数错误
共运行了4次，成功2次，失败2次
```

我们一共加了四个注解，然后 抛了两个异常，如结果所示，说明我们设置的注解起了作用。

