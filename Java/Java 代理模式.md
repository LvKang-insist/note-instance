####  代理模式 是Java 的一种设计模式，主要就是在访问目标对象的时候通过一个代理类 来访问目标对象。那这样做有什么好处吗？

##### 使用代理模式 可以在通过代理类访问目标对象的基础上增加一些额外的功能。比如说目标对象只有一个保存数据的方法，我现在要保存一些数据，但是我想在保存数据的时候删除原来的数据。这样我就必须修改目标对象中的方法。但是有了代理以后我可以让代理类去完成这件事，目标对象只保存数据，其他的都让代理类来做。简单的说就是可以通过代理类来扩展目标对象的方法。

---
- 静态代理

```java
//java 静态代理
public class Demo1 {
	public static void main(String[] args) {
		//创建真目标对象
		Person per1 = new Person();
		//创建代理人，把真目标对象传进去，形成代理关系
		Agent agent = new Agent(per1);
		agent.save();//通过代理人 执行目标对象的方法

	}
}

//接口
interface Marry{
	void save();
}

//目标对象
class Person implements Marry{
	@Override
	public void save() {
		System.out.println("我是目标对象：我在保存数据");
	}
}
//代理人
class Agent implements Marry{
	
	Marry m ;
	Agent(){};
	Agent(Marry m){
		this.m = m;
	}
	
	@Override
	public void save() {
		System.out.println("代理类：在保存数据前 删除原来的数据");
		m.save();
		System.out.println("代理类：保存成功");
		
	}
}
```

结果如下：

```
代理类：在保存数据前 删除原来的数据
我是目标对象：我在保存数据
代理类：保存成功

```

使用静态代理 有三个条件，1,有一个接口 2，有目标对象和代理人 3，目标对象和代理人必须实现同一个接口。

使用静态代理的好处：只要实现接口，就可以对目标对象进行扩展，保证的目标对象的不变性。

缺点：如果有多个地方需要保存数据，而且保存数据的时候需要让代理做其他事，这些事情如果不一样，就需要很多代理类。并且只要接口中增加方法，目标对象和代理类都要进行维护。

---
- 动态代理

java 的动态代理中，有一个类Proxy类和InvocationHandler接口是非常重要的。这两个是事项动态代理必须用到的。

proxy 是用来创建动态代理类对象的，有了对象才可以调用目标对象的方法。

InvocationHandler：这个接口是个动态代理类实现的，当通过代理对象调用 目标对象的方法的时候，这个调用就会传递到InvocationHandler接口的invoke(Object proxy, Method method, Object[] args)方法中。
代理对象 就会作为proxy传入，参数Method代表着我们具体调用的那个方法，args为这个方法的参数。


创建代理类对象，调用Proxy的newProxyInstance方法就可以返回一个代理类对象。这个方法有三个参数
- ClassLoader loader：返回类的类加载器。
- Class<?>[] interfaces ：如果此对象表示一个类，则返回值是包含表示该类实现的所有接口的对象的数组。如果此对象表示接口，则该数组包含表示接口扩展的所有接口的对象。
- InvocationHandler h ：表示当前的InvocationHandler实例对象，这是一个接口，可以直接实现，也可以通过匿名内部类实现。代理对象调用方法后，这个接口的invock就会得到执行。


下面 来看一下具体操作

```
public class Demo2 {
	public static void main(String[] args) {
		//创建目标对象
		Data data = new Data();
		//传入目标对象的实例返回代理类
		Save save = new A(data).getProxy();
		//通过代理对象调用目标对象方法，实际上会调用到invoke方法。
		String result = save.save();
		System.out.println(result);
	}
}

//接口
interface Save{
	String save();
}
//目标对象
class Data implements Save{
	@Override
	public String save() {
		System.out.println("我是目标对象，我在保存数据");
		return "我是返回值";
	}
}

//通过过场景类生成代理类
class A{
	
	private Save save ;
	//传入目标类对象
	public A(Save save){
		this.save = save;
	}
	
	public Save getProxy(){
		//返回代理类的对象。
		return (Save)Proxy.newProxyInstance(save.getClass().getClassLoader(),
				save.getClass().getInterfaces(), new InvocationHandler() {
			
			@Override
			public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
				
				System.out.println("代理对象类名："+proxy.getClass().getName());
				System.out.println("调用的方法"+method.getName());
				
				//判断调用的方法
				if(method.getName().equals("save")){
					//通过反射调用save方法
					return method.invoke(save);
				}
				return null;
			}
		});
	}
}
```
结果如下：

```
代理对象类名：demo.$Proxy0
调用的方法save
我是目标对象，我在保存数据
我是返回值
```

使用 动态代理不需要创建代理类，只需要调用方法就可以得到代理对象。代理对象不需要实现接口，但是目标对象一定要实现接口。

两者的区别：
- 静态：由程序员创建代理类或特定工具自动生成源代码再对其编译。在程序运行前代理类的.class文件就已经存在了。

- 动态：在程序运行时运用反射机制动态创建而成。