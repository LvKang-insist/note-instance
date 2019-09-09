## 一直在使用泛型，但是对泛型的了解非常浅，所以今天就做一个详细的笔记。

####  Java 泛型（generics）是 JDK 5 中引入的一个新特性, 泛型提供了编译时类型安全检测机制，该机制允许程序员在编译时检测到非法的类型。泛型的本质是参数化类型，也就是说所操作的数据类型被指定为一个参数。



### 1,定义泛型类、泛型接口

```

//泛型类
public class Apple<T>{

    private T info;
    Apple(T info){
        this.info = info;
    }

    public T getInfo() {
        return info;
    }

    public void setInfo(T info) {
        this.info = info;
    }

    public static void main(String[] args){
        Apple<String> apple = new Apple<>("苹果");
        System.out.println(apple.getInfo());
        Apple<Integer> intapp = new Apple<>(52);
        System.out.println(intapp.getInfo());
    }

}


//泛型接口
public interface List<E> {
    void add(E e);
}

```
上面定义了一个带泛型的Apple<T>类，在使用的时候，为T传入实际的值，这样就可以生成如Apple<String>,Apple<Integer>形式的逻辑子类(物理上并不存在)


---


### - 怎么派生带泛型的子类？

当创建了带泛型声明的接口、类之后，可以为该接口或者类创建实现类，或者是派生子类，当使用这些接口、父类时不能再包含形参：例如下面的做法就是错误的： 
    
```
//定义类A继承Apple类，Apple类不能跟泛型形参
public class A extends Apple<T>{}
```

如果想从Apple类派生一个子类则可以修改为如下代码：

```
//使用Apple类时为T穿融入String类型
public class A extends Apple<String>{}
```
调用方法时必须为所有的数据形参传入参数值，与调用方法不同的是，使用接口、类时也可以不为泛型形参传入实际的参数参数，即下面的代码也是正确的

```
//使用Apple类时，没有为T形参传入实际的参数
public class A extends Apple{}
```
像这种使用Apple类时省略泛型的形式被称为原始类型(raw type)

如果从Apple<String> 类派生子类1，则在Apple类中所有使用T类型的地方都将被替换成String类型，如果子类需要重写它的方法，就必须注意这一点。如下所示：

```
public class A extends Apple<String> {
    public A(String info) {
        super(info);
    }

    //正确的重写了父类的方法，返回值
    //与父类Apple<String>的返回值完全相同
    @Override
    public String getInfo() {
        return "重写父类的方法";
    }

    @Override
    public void setInfo(String info) {
        super.setInfo(info);
    }
}
```
如果使用Apple时没有传入实际的类型(即原始的类型)，Java编译器可能发出警告：使用了未检查或不安全的操作----这就是泛型的警告。此时会把形参T当做为Object类型处理。如下所示：

```
//没有使用泛型，默认为Object类型
public class A extends Apple {
    public A(String info) {
        super(info);
    }
    
    @Override
    public Object getInfo() {
        return super.getInfo();
    }
    @Override
    public void setInfo(Object info) {
        super.setInfo(info);
    }
}
```


---

### 类型通配符

当我们是用一个泛型类时，都应该为这个泛型类传入一个类型实参。如果没有传入类型实际参数编译器就会提出泛型警告。如下所示：

```
  public void test(List list){
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
```

上面程序当然没有问题，这是一段最普通的遍历集合的代码，问题是List是一个有泛型声明的接口，此处使用没有传入实际的参数，这将引起泛型的警告。所以要为List接口传入时间的类型参数，因为List集合中的元素类型是不确定的，所以讲上面方法改为如下形式。

```
 public void test(List<Object> list){
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
```
表明看起来没有问题，这个方法的声明确实没有任何问题。问题是调用该方法传入的实际参数值可能不是我们期望的。如下：

```
 public static void main(String[] args){
        List<String> list = new ArrayList<>();
        test(list);
    }
```
编译上面的程序，将发生如下错误：

```
不兼容的类型: List<String>无法转换为List<Object>
```
表明List<String>对象不能被当成List<Object>来使用。



---

#### 使用类型通配符


为了表示各种泛型List的父类可以使用类型通配符<?>，将一个问号作为他的参数类型传给list集合，写作List<?>，意思是元素类型为未知的Object，这个问号被称为 通配符 ，他的元素可以匹配任何类型。如上面的可以写成如下形式：
```
 public static void test(List<?> list){
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
```

现在使用任何类型的List来调用它，都可以访问集合list中得元素，其类型时候Object，这个永远是安全的，不管List的真实数据是什么，它包含的都是Object。这种写法可以适应于任何支持泛型声明的接口和类。

但这种带通配符的List仅表示他是各种泛型的List的父类，并不能将元素加入到其中，如下就会引起编译错误：

```
  List<?> list = new ArrayList<>();
        //下面将会引起编译错误
        list.add(new Object());
```

因为程序无法确定list集合中的元素类型，所以不能向其中添加对象。


---

### 类型通配符的上限

当我们定义某个方法的时候，如果想要让这个方法被特定的类调用，就可以使用这种形式。如下所示：

```
     public static void test(List<? extends Apple<String>> list) {
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
```
当我们调用这个方法的时候，传入的参数必须是继承自Apple的，

```
public class A extends Apple<String> {
    public A(String info) {
        super(info);
    }
    
    public static void test(List<? extends Apple<String>> list) {
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
    public static void main(String[] args) {
        List<A> list = new ArrayList<>();
        test(list);
    }
}

```
如上，list<? extends Apple<String>>是一个受限制的通配符的例子，此处的 ？代表是一个未知的类型，就想前面看到的通配符一样，但此处的未知类型一定是Apple的子类型(也可以是 Apple本身)，因此把这种称为通配符的上限。

由于这个程序无法确定这个受限通配符的类型是什么，所以不能把Apple对象或者他的子类加入到这个泛型的集合中去，例如，下面的代码就是错误的:

```
public static void test(List<? extends Apple<String>> list) {
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
        //下面引起编译错误
        list.add(new A(""));
    }
```
简而言之，这种指定类型通配符上限的集合，只能从集合中取元素(取出的元素是总上限的类型)，不能像集合添加元素。因为编译器无法确定这集合中的元素是那种类型。


---

### 类型通配符的下限

通配符下限用<? super 类型> 的方式来指定，和通配符上限的作用恰好相反.

```
public class A extends Apple<String> {
    public A(String info) {
        super(info);
    }

    public static void test(List<? super A> list) {
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
    }
    public static void main(String[] args) {
        List<Apple<String>> list = new ArrayList<>();
        test(list);
    }
}
```
修改一下text方法，改为 List<? super A> ，这个表示这个list集合里面存的只能是A，或者是A的父类。向下限定可以向其中添加元素，但是添加的类型必须是 super 后面跟的类型，如下所示：

```
public class A extends Apple {
    public A(String info) {
        super(info);
    }
    public static void test(List<? super A> list) {
        for (int i = 0; i < list.size(); i++) {
            System.out.println(list.get(i));
        }
        //可以添加
        list.add(new A(""));
        //编译报错
//        list.add(new Apple<String>(""));
    }
    public static void main(String[] args) {
        List<Apple> list = new ArrayList<>();
        test(list);
    }
}
```


---

#### 泛型形参的上限

java泛型不仅允许在通配符上使用上限，而且可以在定义泛型形参的时候设定上限，表示传给该泛型形参必须是该类上限类型，要不就是该上限类型的子类，如下所示：

```
public class Apple<T extends Number>{
    private T info;
    Apple(T info){
        this.info = info;
    }

    public T getInfo() {
        return info;
    }

    public void setInfo(T info) {
        this.info = info;
    }
    public static void main(String[] args){
        
        Apple<Double> dou = new Apple<>(2.2);
        Apple<Integer> integer = new Apple<>(2);
        
        //下面将编译失败，视图将String类型传给形参T
        //是String不是Number的子类，所以编译错误
        Apple<String> apple = new Apple<>("苹果");
        
    }
}
```
String既不是Number类型，也不是他的子类，所以报错.


在一种更极端的情况下，程序需要为泛型设置多个上限(至少有一个父类上限，可一个有多个接口上限)，表明该泛型形参必须是父类的子类(父类也行),并且实现多个上限的接口，如下所示：

```
//表明T类型必须是Number 或 是其子类，并且必须实现 Runnable接口
public class Apple<T extends Number & Runnable>{}
```

与类同时继承父类，实现接口类似的是，为泛型形参设置多个上限的时候，所有的接口上限必须位于类上限之后。也就是说，如果需要为泛型形参指定类上限，类上限必须放在第一位。

---


### 泛型方法

如下所示：

```
 //泛型方法，该方法中带有一个T泛型形参 
    public static <T> void copy(List<T> from, List<T> to) {
        for (T t : from) {
            to.add(t);
        }
    }
    public static void main(String[] args){
        
        List<Integer> list1 = new ArrayList<>();
        List<Number> list2 = new ArrayList<>();
        copy(list1,list2);//报错：编译器无法确定泛型T的类型。
        
        List<String> list3 = new ArrayList<>();
        List<String> list4 = new ArrayList<>();
        copy(list3,list4);//泛型T 为String 类型
        
    }
```
copy方法中带有一个带T的泛型形参，但是在调用的时候 传的泛型参数为String，Integer类型，编译器无法准确的推断出泛型方法中泛型形参的类型，所以会报错.

为了避免上面的错误，可将代码改为如下格式.

```
    //泛型方法，该方法中带有一个T泛型形参
    public static <T> void copy(List<? extends T> from, List<T> to) {
        for (T t : from) {
            to.add(t);
        }
    }
    public static void main(String[] args){

        List<Integer> list1 = new ArrayList<>();
        List<Number> list2 = new ArrayList<>();
        copy(list1,list2);//报错：编译器无法确定泛型T的类型。

        List<String> list3 = new ArrayList<>();
        List<String> list4 = new ArrayList<>();
        copy(list3,list4);//泛型T 为String 类型

    }
```
这种采用了类型通配符的方式，只要copy方法中的前一个集合的元素类型是最后一个集合元素的子类即可。


---

### 带泛型的返回值


```
    //该返回值表明这个方法的返回值是一个list集合，并且元素的类型为T
   public static <T>  List<T> copy( List<T> to) {
        List<T> list = new ArrayList<>();
        for (int i = 0; i < to.size(); i++) {
            list.add(to.get(i));
        }
        return list;
    }

    //返回值为T
    public static <T extends Number> T  test(T t){
        return t;
    }

    public static void main(String[] args){

        List<Integer> list1 = new ArrayList<>();
        //该方法复制了一个新的集合,返回值为list集合 返回值为泛型
        List<Integer> copy = copy(list1);

        
       Double d = new Double(2.0);
       Double test1 = test(d);//返回值为传入的类型

    }
```

> 参考自 疯狂java讲义


