
 @CallSuper注解主要是用来强调在覆盖父类方法的时候，需要实现父类的方法，及时调用对应的super.**方法，当使用 @CallSuper 修饰了某个方法，如果子类覆盖父类该方法后没有实现对父类方法的调用就会报错，如下所示：
 

```
class NULL {
    @CallSuper
    protected void Body(){
        System.out.println("Null_Body");
    }
}
class A extends NULL{
    protected void Body(){
        super.Body();
        System.out.println("A_Body");
    }
}
```

当使用@CallSuper 修饰某个方法后，子类覆盖该方法的时候就必须使用super.父类的该方法。


---
注意：java中是没有这个注解的。

 