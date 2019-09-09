## 什么是反射：



对于任何一个类，我们都能知道他有哪些方法。对于任何一个对象我们都可以调用他的任意一个方法和属性，这种动态的获取信息以及动态的调用对象的方法就称为java的反射机制。形象一点说，任何一个类或者对象，对我们来说都是透明的，想要啥直接拿就可以。


---

#### 首先列出我们要反射的类：

```
public class Person {
	
	int age;
	String name;
	private String email;
	
	public Person(int age, String name) {
		this.age = age;
		this.name = name;
	}
	
	public Person(int age){};
	public Person(){};
	
	
	private String getEmail() {
		System.out.println("我是getEmail");
		return email;
	}

	private void setEmail(String email) {
		this.email = email;
	}
	public int getAge() {
		return age;
	}

	public void setAge(int age) {
		this.age = age;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		System.out.println("我是setName");
		this.name = name;
	}
	
}
```



##### 要想对一个类进行反射，就必须获取他的字节码字节码。有三种形式：

```
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

#### 通过反射获取类中的所有构造器


```
        //获取类中的所有构造器
    	Constructor<?>[] con = c1.getConstructors();
    	System.out.println("共"+con.length+"构造器");
    	for (int i = 0; i < con.length; i++) {
    		System.out.println("第"+i+"个构造参数");
    		//获取构造器参数类型
			Class[] paramenter = con[i].getParameterTypes();
			for (int j = 0; j < paramenter.length; j++) {
				//打印构造器参数
				System.out.print(paramenter[j].getName()+"   ");
			}
			System.out.println();
		}
```
结果如下：

```
共3构造器
第0个构造参数

第1个构造参数
int   
第2个构造参数
int   java.lang.String   
```
一个无参，还有两个有参构造。


---
#### 通过获取的构造器进行实例化

```
    try {
    		Person per1 = (Person) con[0].newInstance();
    		Person per2 = (Person) con[1].newInstance(20);
    		Person per3 = (Person) con[2].newInstance(19,"lv");
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} 
```

---
#### 通过反射获取类中的属性


```
Person per1 = null,per2 = null,per3= null;
    	try {
    		 per1 = (Person) con[0].newInstance();
    		 per2 = (Person) con[1].newInstance(20);
    		 per3 = (Person) con[2].newInstance(19,"lv");
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} 
    	
    	//获取某个对象内的某个字段
    	try {
			Field f = c1.getDeclaredField("email"); //或取私有属性
			//对私有字段的访问取消检查
			f.setAccessible(true);
			//修改该对象的emal字段
			f.set(per3, "emal@qq.com");
			
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
    	
    	
    	//获取当前类的所有属性类型和值
    	Field[] field = c1.getDeclaredFields();
    	//对私有字段的访问取消检查
    	Field.setAccessible(field, true);
    	for (int i = 0; i < field.length; i++) {
			try {
				System.out.println("属性名:"+field[i].getName()+"属性类型:"
						+field[i].getType()+"属性值:"+field[i].get(per3));
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
```
结果如下：

```
属性名:age属性类型:int属性值:19
属性名:name属性类型:class java.lang.String属性值:lv
属性名:email属性类型:class java.lang.String属性值:emal@qq.com
```
注意：取消安全性检查后才可以访问私有属性

---


#### 通过反射获取类中的方法并运行

```
        //获取全部方法
    	Method[] methods = c1.getDeclaredMethods();
    	for (int i = 0; i < methods.length; i++) {
			System.out.println("方法名称："+methods[i].getName()+
					" 返回值类型 ："+methods[i].getReturnType()+
					" 方法参数："+methods[i].getParameterTypes());
			
		} 
    	
    	//获取无参私有方法
    	try {
			Method method = c1.getDeclaredMethod("getEmail");
			//对私有字段取消安全检查
			method.setAccessible(true);
			System.out.println("私有方法名称："+method.getName()+" 返回值类型："
			+method.getReturnType().getName());
			//调用无参的私有方法
			method.invoke(per3);//即getEmail方法
			
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
    	
    	//获取有参方法
    	try {
			Method meth = c1.getDeclaredMethod("setName", String.class);
			//调用有参方法
			meth.invoke(per3, "我是Name");//即setName方法
		} catch (Exception e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} 
    	
```
结果如下：

```
方法名称：main 返回值类型 ：void 方法参数：[Ljava.lang.Class;@6d06d69c
方法名称：getName 返回值类型 ：class java.lang.String 方法参数：[Ljava.lang.Class;@7852e922
方法名称：setName 返回值类型 ：void 方法参数：[Ljava.lang.Class;@4e25154f
方法名称：setEmail 返回值类型 ：void 方法参数：[Ljava.lang.Class;@70dea4e
方法名称：getAge 返回值类型 ：int 方法参数：[Ljava.lang.Class;@5c647e05
方法名称：setAge 返回值类型 ：void 方法参数：[Ljava.lang.Class;@33909752
方法名称：getEmail 返回值类型 ：class java.lang.String 方法参数：[Ljava.lang.Class;@55f96302
私有方法名称：getEmail 返回值类型：java.lang.String
我是getEmail
我是setName
```


---
总结一下：
 这些 只是最基本的反射，更多的使用方法还是需要去查api。其实反射是非常重要的。我们还是有必要去掌握它。
 
 
 
