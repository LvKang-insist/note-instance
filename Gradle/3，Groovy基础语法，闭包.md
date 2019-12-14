### 基本类型

​		其实在 Groovy 中是没有基本类型的，例如下所示：

```groovy
int x = 10;
println(x.class)
```

​		打印结果如下：

```
class java.lang.Integer
```

​		可以看到他是一个 Integer 对象。其实在底层编译的时候已经将 int 装箱成了 对象。

### 变量的定义

​		groovy 定义变量有两种方式，分别是 强类型定义 和 若类型定义，强类型定义和 java 中的意义，但是弱类型定义是什么呢，如下：

```
//强类型定义方式
int x = 10;
println(x.class)

//弱类型定义方式
def x1 = 1;
println(x1.class)
def x2 = 3.14;
println(x2.class)
def x3 = "Android"
println(x3.class)
```

​	结果如下：

```
class java.lang.Integer
class java.lang.Integer
class java.math.BigDecimal
class java.lang.String
```

​	第二行 是大数据类型

​	其实弱类型定义就是不指定具体的类型。通过值来确定它到底是什么类型。用 def 来定义

​	如何选择呢？如果在本类中使用，则建议使用 def 类型。但是别的类也要用到这个变量。则尽量使用强类型定义；**def 声明的变量类型可以随时改变**，如 x1 是 Integer 。如果给 x1 赋值为 String，这样他就会变为 String。正是这个原因所以才不建议在 别的类使用这个变量时定义为 def。

### 字符串

​	常用的定义方式：

```groovy
//1,通过 单引号定义String ，和双引号没有任何区别 ,转义 '单引号 \'String \''
def name = '单引号 String'
println(name)


//2,通过三个单引号定义String ,但定义的格式进行输出，
def name1 = '''第一行
第二行
第三行'''
println(name1)

// 3,双引号 ,可扩展字符串，扩展后使用的不是String，而是GString类
def name2 = "双引号"
def hello = "hello ${name2}"
println(name2)
println(hello.class)

//可扩展字符串使用 ,可以扩展任意的表达式
def sum = "sum = ${5 - 6 - 6 + 12}"
println(sum)

def result = echo(sum)
println(result)
String echo(String message){
    return message;
}

```

通过上面这几种方式可以 定义字符串，其实一般最常用的就是 第三种了。注意扩展后的字符串就不是 String。而是 GString。不信的话可以打印看一下。

### 字符串方法

```
字符串的方法
1,第一种就是String类原有的方法
2,第二种则是 DefaultGroovyMethods ，是 Groovy 对所有对象的扩展
3,第三种是 StringGroovyMethods ，继承自 DefaultGroovyMethods 。
```

重点要看的是 StringGroovyMethods 。他继承自第二种，第一种是 String，比较熟悉。

首先定义一个字符串：

```groovy
String str = "哈哈哈"
```

- 填充

  ```groovy
  println str.center(8, 'a') //填充: aa哈哈哈aaa
  println str.padLeft(8, 'a') //填充：aaaaa哈哈哈
  ```

- 比较

  ```groovy
  println str > "嘻嘻嘻" //通过运算符比较字符串大小 ：false
  ```

- 索引

  ```groovy
  println str.getAt(0) //下标为0 ：哈
  println str[0]  //通过类似于数组的方式来获取 ：哈
  println str[0..1] //指定范围 :哈哈
  ```

- 减法

  ```groovy
  println str.minus("哈哈") //减法 : 哈 	--减去两个剩一个 。
  println str - "哈哈";
  ```

- 倒序

  ```
  println str.reverse()
  ```

- 首字母大写

  ```
  println str.capitalize()
  ```

- 判断 str 是否为数字类型

  ```groovy
  println str.isNumber()
  ```

- 转为基本数据类型

  ```groovy
  println str.toInteger()
  println str.toBoolean()
  println str.toDouble()
  ```

- 其实方法还有很多，这里就不一一列举了。

  [可查看文档](http://www.groovy-lang.org/documentation.html)

### 控制语句

​	switch：groovy 中的 switch 和 java 中的有明显的不同，它可以接受任意的数据类型。如下：

```groovy
// switch ,可以传入任意类型的数据
def x = 1.23
def result;
switch (x) {
    case '哈哈哈哈':
        result = '0'
        break
    case '嘻嘻':
        result = '1'
        break
    case [4, 5, 6, 'inlist']: //列表
        result = '2'
        break
    case 12..30: //范围
        result = '3'
        break
    case Integer:
        result = '4'
        break
    case BigDecimal:
        result = '5'
        break
    default:
        result = 'defalut'
        break
}
println result //5
```

​	for 循环

```groovy
// 对范围的 for 循环
def sum = 0
for (i in 0..9) {
    sum += i
}
println sum

//对 list 进行循环
sum = 0
for (i in [1, 2, 3, 4, 5, 6, 7, 8, 7]) {
    sum += i
}
println sum

//对 map 进行循环
sum = 0;
for (i in ['one': 1, 'two': 2, "three": 3]) {
    sum += i.value
}
println sum; 
// 45
// 43
// 6
```

### 闭包

​		**基础详解**

​			闭包是一个开放的，匿名的代码块，可以带参数，返回一个值。可以通过一个变量引用到他，也可以传递给别人进行处理。

- 闭包的创建和调用

  ```groovy
  def clouser = {
      println "hello groovy"
  }
  //两种方式调用闭包
  clouser.call()	// hello groovy
  clouser()		// hello groovy
  ```

- 创建带参数的闭包和调用

  ```groovy
  def clouser = {
          //箭头前面是参数，后面是闭包体
      String name, int age ->
          println "hello groovy ${name} -- ${age}"
  }
  //调用带参数闭包
  clouser.call("张三", 32)  // hello groovy 张三 -- 32
  clouser("李四", 23)		//hello groovy 李四 -- 23
  ```

- 默认参数

  ```groovy
  //每一个闭包都有一个默认的 it 参数，如果不想使用，则显示的定义即可。
  def clo = {
      println "hello ${it}"
  }
  clo.call("李四")	 //hello 李四
  ```

- 闭包返回值

  ```groovy
  def clouser = {
          //箭头前面是参数，后面是闭包体
      String name, int age ->
          return "hello groovy ${name} -- ${age}"
  }
  //调用带参数闭包
  println clouser.call("张三", 32) //hello groovy 张三 -- 32
  ```

​		**使用详解**

​		基本类型结合使用：

------



- 阶乘

  ```groovy
  int x = 5
  println fab(x)
  //求 number 的阶乘
  int fab(int number) {
      int reslut = 1
      //调用 upto 方法，传入 number 和 闭包 ，闭包中有一个 num 参数
      1.upto(number, { num -> reslut *= num })
      return reslut
  }
  //结果：120
  ```

  为什么要这样写呢？看一下源码秒懂！

  ```groovy
  //打眼一看这里需要三个参数，我们只传了两个？，注意：1.upto()中的 1，代表的是第一个参数，
  public static void upto(Number self, Number to, @ClosureParams(FirstParam.class) Closure closure) {
  		//拿到开始的位置和结束的位置
          int self1 = self.intValue();
          int to1 = to.intValue();
          //小于等于
          if (self1 <= to1) {
              for (int i = self1; i <= to1; i++) {
              	//调用传入的闭包，将i 传进去，从这里就可以知道我们写的闭包为啥要一个 num 的参数了。
                  closure.call(i);
              }
          } else
              throw new GroovyRuntimeException("The argument (" + to +
                      ") to upto() cannot be less than the value (" + self + ") it's called on.");
      }
  ```

- 阶乘2

  ```java
  int fab2(int number) {
      int result = 1
      number.downto(1) {
          num -> result *= num
      }
      return result
  }
  //结果：120
  ```

  你知道 downto 方法传入了几个参数吗。。。。没错，是3个，number是一个，1 是一个，括号外面的闭包是一个。闭包写在括号外面和写在里面效果完全一样。这个是倒着来的。具体的可以查看源码。

- 累加

  ```groovy
  int cal(int number) {
      int result
      // times 只有两个参数，所以可以使用这种方式来写 ，具体的实现可以看源码，非常简单
      number.times {
          num -> result += num
      }
      return result;
  }
  ```

字符串的结合使用

------

定义一个字符串：

```groovy
String str = "the 2 and 3 is 5"
```

- each 遍历

  ```groovy
  str.each {
      String temp ->
          print temp  //the 2 and 3 is 5
  }
  //看一下 each 方法的源码
  public static <T> T each(T self, Closure closure) {
      	//第一个是迭代器，第二个则是闭包了
          each(InvokerHelper.asIterator(self), closure);
          return self;
      }
  
  public static <T> Iterator<T> each(Iterator<T> self, @ClosureParams(FirstParam.FirstGenericType.class) Closure closure) {
      	//遍历 字符串，
          while (self.hasNext()) {
              Object arg = self.next();
              //调用闭包，传入了一个参数，从这里就可以知道我们在定义闭包的时候需要加一个参数
              closure.call(arg);
          }
          return self;
      }
  ```

- 查找符合条件的**第一个**

  ```groovy
  println str.find {
      String s ->
      	//返回 boolean 。是否符合条件，这里的条件为：是否是数字
          s.isNumber() 
  }
  //结果：2 ，第一个符合结果的是2，看一下源码
      public static Object find(Object self, Closure closure) {
          //将闭包弄成了 boolean 类型的
          BooleanClosureWrapper bcw = new BooleanClosureWrapper(closure);
          //迭代
          for (Iterator iter = InvokerHelper.asIterator(self); iter.hasNext();) {
              Object value = iter.next();
              //调用闭包，如果返回 true。则表示满足条件，接着就将 value 返回，然后就退出了。不在执行
              if (bcw.call(value)) {
                  return value;
              }
          }
          return null;
      }
  
  //查找全部
  println str.findAll {
      String s ->
          return s.isNumber()
  }
  //结果：[2, 3, 5] ，查看源码可知找到后会存在集合中，最后进行返回
  ```

- 判断是否包含数字

  ```java
  println str.any {
      String s ->
          return s.isNumber() // true
  }
  //源码可自行查看
  ```

- 字符串是否每一项都为数字

  ```groovy
  println str.every {
      String s ->
          s.isNumber() //false
  }
  //源码
  public static boolean every(Object self, Closure predicate) {
        return every(InvokerHelper.asIterator(self), predicate);
  }
  public static <T> boolean every(Iterator<T> self, @ClosureParams(FirstParam.FirstGenericType.class) Closure predicate) {
          BooleanClosureWrapper bcw = new BooleanClosureWrapper(predicate);
          while (self.hasNext()) {
              //是否满足闭包中的条件
              if (!bcw.call(self.next())) {
                  return false;
              }
          }
      	//循环完成后表示都满足条件，返回true
          return true;
      }
  ```

- 将字符串转为集合

  ```groovy
  println str.collect {
      //将 字符转换大写
      it.toUpperCase()
  }
  //查看源码
   public static <T> List<T> collect(Object self, Closure<T> transform) {
       	//这里创建了一个新的集合
        return (List<T>) collect(self, new ArrayList<T>(), transform);
   }
  public static <T> Collection<T> collect(Object self, Collection<T> collector, Closure<? extends T> transform) {
      	//给字符串加上迭代器
          return collect(InvokerHelper.asIterator(self), collector, transform);
  }
    public static <S,T> Collection<T> collect(Iterator<S> self, Collection<T> collector, @ClosureParams(FirstParam.FirstGenericType.class) Closure<? extends T> transform) {
          while (self.hasNext()) {
              //调用闭包中的方法，将返回的结果保存到集合中
              collector.add(transform.call(self.next()));
          }
        	//返回集合
          return collector;
      }
  ```

  

​	**进阶详解**

- 闭包关键变量	this，owner，delegate

  this：闭包定义处的类

  owner ：代表闭包定义处的类或者对象。因为闭包是可以嵌套的，当闭包嵌套的时候，owner 就是闭包，而不是所处的类

  delegate：任意一个对象，默认与 owner 一样

  看一个例子：

  ```groovy
  def clouser = {
      println this  //闭包定义处的类
      println owner  //代表闭包定义处的类或者对象，因为闭包是可以嵌套的,如果 闭包中有嵌套，则 owner 代表的就是闭包
      println delegate //任意一个对象。默认与 owner 一致
  }
  clouser.call()
  //结果：
  variable.ClooseStudy@5acf93bb
  variable.ClooseStudy@5acf93bb
  variable.ClooseStudy@5acf93bb
  ```

  可以看到这三个结果是一样的。因为这里的闭包没有嵌套，所以 owner 是定义闭包处的类，delegate 则和 owner 一致

  接着看：

  ```groovy
  //定义了一个内部类
  class PerSon {
      //静态闭包
      def static classClouser = {
          println "Class  " + this
          println "Class  " + owner
          println "Class  " + delegate
      }
  	//静态方法
      def static say() {
          def classClouser = {
              println "method  " + this
              println "method  " + owner
              println "method  " + delegate
          }
          classClouser.call()
      }
  }
  //结果:
  Class  class variable.PerSon
  Class  class variable.PerSon
  Class  class variable.PerSon
  method  class variable.PerSon
  method  class variable.PerSon
  method  class variable.PerSon
  ```

  其实这个和上面的都差不多，这里的 this 就是内部类了，而 owner 因为不是嵌套，所以也是 Person。delegate 就不用说了。注意：这里的结果后面没有内存地址是因为 闭包和方法都是静态的。

  接着看：

  ```groovy
  //闭包中定义一个闭包
  def nestClouser = {
      def innerClouser = {
          println "innerClouser  " + this
          println "innerClouser  " + owner
          println "innerClouser  " + delegate
      }
      innerClouser.call()
  }
  nestClouser.call()
  //结果：
  innerClouser  variable.ClooseStudy@6cd28fa7
  innerClouser  variable.ClooseStudy$_run_closure2@f0c8a99
  innerClouser  variable.ClooseStudy$_run_closure2@f0c8a99
  ```

  这里的 this 是当前类，但是 owner 不一样了，可以看到他在后面拼接了一点地址，这个就是外层闭包的对象。delegate 默认和 owner 一样，所以他们两个一样。你可以打印一下 nestClouser 的地址看一下是否和 owner 一样。。

  如果修改了 delegate 的值，则他和 owner 就不一样了。注意：this 和 owner 是不能被修改的，只有 delegate 可以修改

### 闭包委托策略

​		首先看一段代码

```java
class Student {
    String name
    def pretty = {
        "My name is ${name}"
    }

    String toString() {
        pretty.call()
    }
}


class Teacher {
    String name
}

def student = new Student(name: '张三')
def teacher = new Teacher(name: "老师")
println student.toString()
```

​	定义了两个类，一个学生和一个老师。

​	如果调用 student 的 toString 方法，会打印什么信息呢？

```
My name is 张三
```

​	可以看到打印的是张三。那如果要打印 Teacher 的 name 呢？这个时候就要用到 委托策略了

```groovy
student.pretty.delegate = teacher
println student.toString()
```

​	将 teacher 传给 student 的闭包的 delegate  。这个时候在输出一下，你会发现没有任何改变。。。

​	我们少改了一个东西， 默认的委托策略是 OWNER_FIRST ,即从 owner 中开始寻找对象 。owner 指向的是当前类 student。所以才没有发生任何变化。所以我们要修改委托策略：

```groovy
student.pretty.delegate = teacher
student.pretty.resolveStrategy = Closure.DELEGATE_FIRST //修改委托策略
println student.toString()
//My name is 老师
```

​	修改为 DELEGATE_FIRST 后会从 delegate 中开始寻找，因为上面指定了 delegate 是 teacher ，所以输出就变了 

​	修改 Teacher 中的 name 字段名字，如下：

```java
class Teacher {
    String name1
}
def teacher = new Teacher(name1: "老师")
```

​	这个时候在运行会发现 运行的结果是 张三 ，因为 student 中的闭包里面用的是 name，而不是 name1，所以找不到 name 后就会寻找当前类的 name。 

​	修改一下委托策略，如下：

```groovy
student.pretty.resolveStrategy = Closure.DELEGATE_ONLY
```

​	接着在运行一下会报错：

```
name for class: variable.Teacher Possible solutions: name1
```

​	在 Teacher 中 没有找到 name ，所以报错。这个策略只会从 delegate 中寻找，而不会从当前类中找

​	委托策略一共有四种：

```groovy
public static final int OWNER_FIRST = 0;
public static final int DELEGATE_FIRST = 1;
public static final int OWNER_ONLY = 2;
public static final int DELEGATE_ONLY = 3;
```

​	默认的就是 OWNER_FIRST ，即从 owner 中开始寻找。

### 列表

```groovy
//列表
def list = [1, 23, 324, -5, 6]  // groovy 中定义的方式，和上面的完全一样 ，并且添加元素。
println list.class

//排序
list.sort()
println list
list.sort {
    a, b ->
        a > b ? 0 : -1
}
println list
//按字符串长度排序
def strList = ['a', 'ab', 'abc', 'aaaaa', 'b', 'cadfa', 'ace']
strList.sort {
    it ->
        return it.size()
}
println strList

//查找被 2 整除
println list.findAll() {
    return it % 2 == 0
}
// 判断是否列表中元素是否都能够被 2 整除
println list.every {
    return it % 2 == 0;
}
println list.min() //最小值
println list.max() //最大值
println list.min { Math.abs(it) } //绝对值 最小值
println list.max { Math.abs(it) } //绝对值 最小值

println list.count {
    return it % 2  //统计，有多少偶数
}

//添加
list.add(5)
list.leftShift(4)
println list + [4, 5]

//删除
list.remove(0) //删除下标为 0 的元素
println list
list.remove((Object) 4) //删除内容为 4 的元素
println list
println list.removeAt(0) //删除指定位置元素，并返回此元素
list.removeElement(0) //删除
println list - [1, 23] //删除 1 和 23

```

上面是列表的定义和常用的方法

有没有感觉和数组的定义一样。当然数组的定义方式也变了，如下：

```groovy
//数组
def array = [1, 3, 4, 5] as int[]   
// groovy 中 数组的定义
int[] array2 = [13, 4, 5, 6]  //强类型数组定义
```

### Map

```groovy
//定义
def map = [red: '红色', yellow: '黄色', black: '黑色']

//查找
println map.get('red')
println map['yellow']
println map.black
println map.find {
    it.getValue().equals("红色")
}
//计数
println map.count {
    it.value.equals('黄色')
}
println map.findAll {
    return it.key.equals('red') //key 为 red
}.collect {
    it.value  //key 为 red 的value
}

//分组
println map.groupBy {
    return it.value.equals('红色') ? '红色' : '其他颜色'
}


//添加
map.put('blue', '绿色')
map.green = '灰色'
map.data = [a: 0, b: 1];
println map
println map.getClass() //默认是 LinkerHash ， 在定义的时候 通过 as HashMap 可转为 HashMap

//遍历
map.each {
    def m ->
        println m.key + "----" + m.value
}
//带下表的变量
map.eachWithIndex { Map.Entry<String, String> entry, int i ->
    println "index ${i}" + entry.key + "----" + entry.value
}
map.each {
    key, value ->
        println key + "----" + value
}

//排序
println map.sort()
println map.sort {
    s1, s2 ->
        return s1.key < s2.key ? 0 : -1
}
```

以上为最常见的使用

### 范围

```groovy
//定义
def range = 1..29

println range[0]//获取
println range.contains(15) //是否包含
println range.from //开始
println range.to    //结束

//遍历
range.each {
    print it
}
println ""
for (i in range) {
    print i
}
println ""

//在 switch 中使用
def sh = {
    int x ->
        def result = 0;
        switch (x) {
            case 0..10:
                result = 1;
                break
            case 10..20:
                result = 2;
                break
            case 20..30:
                result = 3;
                break
        }
        return result;
}
println sh.call(20)
```

```groovy
public interface Range<T extends Comparable> extends List<T> {
	......
}
```

其实 Range 是 继承自 List。所以 List 有的方法他都有。

