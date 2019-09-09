Lambad表达式永续使用更简洁的代码来创建只有一个抽象方法的接口(这种接口被称为函数式接口)的实例。

1，下面看一下简单的用法。

```
//定义一个接口
interface onReqeust {
    int onSum(int a,int b);
}

//普通的实现方法
onReqeust onReqeust = new onReqeust() {
            @Override
            public int onSum(int a, int b) {
                return a+b;
            }
        };
        System.out.println(onReqeust.onSum(2,4));

//使用Lambad表达式之后：
onReqeust on = (a,b)->{
            int sum = a+b;
            return sum;
        };
        System.out.println(on.onSum(2,4));

//如果要要实现的方法只有一行代码，则可以这样写:
onReqeust on = (a,b)->a+b;
        System.out.println(on.onSum(2,4));
        
```
从上面可以看出，使用了Lambad表达式之后，代码变得非常的简单，再也不用new XXX（）这种繁琐的代码了。使用Lambad不需要指出重写方法的名字，也不需要给出重写方法的返回值，只要给出重写方法括号以及括号里面的形参即可。

从上面语法格式可以看出，Lambad由三部分组成
- 形参列表。形参列表允许省略形参类型。如果形参列表只有一个参数，甚至连形参列表的圆括号也可以省略。
- 箭头(->)必须通过英文中划线和大于符号组成。
- 代码块，如果代码块只包含一条语句，Lambad表达式允许省略代码块的花括号，那么就不要用花括号表示语句结束。Lambad代码块只有一条return 语句，甚至可以省略return 关键字。Lambad表达式需要返回值，而他的代码中仅有一条省略了return的语句，Lambad表达式就会自定返回这条语句的值。


下面看一几种简化的写法；

```
interface onReqeust1 {
    void onSum(int a,int b);
}
interface onReqeust2 {
    int onPrint(String s);
}
interface onReqeust3 {
    String onSum(String s1,String s2);
}


public class Demo1 {

    public static void main(String[] args) {

        onReqeust1 o1 = (a,b)-> System.out.println(a+b);
        o1.onSum(3,4);

        onReqeust2 o2 = (s)->{
            int ints= s.length();
            return ints;
        };
        o2.onPrint("我是Lambad表达式");

        onReqeust3 o3  = (s1,s2)->{
            String s = s1+s2;
            return s;
        };
        o3.onSum("我是","o3");
    }
}
```
上面就是几种比较简单的例子。

Lambad表达式与函数式接口
Lambad表达式的类型，也被称为 目标类型 。Lambad表达式的目标类型必须是“函数式接口”,函数式接口可以包含多个默认方法，类方法，但是只能声明一个抽象方法。

java8专门为函数式接口提供了 @FunctionalInterface 注解，该注解通常放在接口接口定义前面，该注解对程序功能没有任何作用，他用于告诉编译器执行更加严格检查该接口必须是函数式接口，否则就会报错。

由于Lambad表达式的结果就是被当成对象，因此程序中完全可以用Lambad表达式进行赋值。


```
//runnable接口只包含了一个无参数的方法
//Lambad表达式代表匿名方法实现了Runnable中的方法。
//因此下面的Lambad创建了一个Rannable的对象。
  Runnable runnable = ()-> {
            System.out.println();
        };
```


Lambad表达式实现的是匿名方法， 因此他只能实现特定函数式接口中的唯一方法。这意味着Lambad有如下两个限制：
- Lambad表达式的目标类型必须是明确的函数式接口
- 表达式只能为函数式接口创建对象。Lambad表达式只能实现一个方法，因此他只能为只有一个抽象方法的接口创建对象。

关于第一点限制，看下面代码是否正确

```
        Object o = ()-> {
            System.out.println();
        };
```
编译代码回报如下错误：

```
错误: 不兼容的类型: Object 不是函数接口
```
从错误信息可以看出，Lambad表达式类型必须是明确的函数式接口。
为了保证Lambad表达式的目标类型是一个明确的函数式接口，可以有如下三种方式
- 将Lambad表达式赋值给函数式接口类型的变量
- 将Lambad表达式作为函数式类型的参数传给某个方法。
- 使用函数式接口对Lambad表达式进行强制类型转换。

因此，只要将上面的代码改为如下形式即可。

```
        Object o = (Runnable)()-> {
            System.out.println();
        };
```
