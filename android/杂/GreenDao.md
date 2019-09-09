1,GreenDao

GreenDao的核心类有三个：分别是DaoMaster,DaoSession,XXXDao，这三个类都会自动创建，无需自己编写创建！

- DaoMaster:：DaoMaster保存数据库对象（SQLiteDatabase）并管理特定模式的DAO类（而不是对象）。它有静态方法来创建表或删除它们。它的内部类OpenHelper和DevOpenHelper是SQLiteOpenHelper实现，它们在SQLite数据库中创建模式。
- DaoSession：管理特定模式的所有可用DAO对象，您可以使用其中一个getter方法获取该对象。DaoSession还提供了一些通用的持久性方法，如实体的插入，加载，更新，刷新和删除。
- XXXDao：数据访问对象（DAO）持久存在并查询实体。对于每个实体，greenDAO生成DAO。它具有比DaoSession更多的持久性方法，例如：count，loadAll和insertInTx。
- Entities ：可持久化对象。通常, 实体对象代表一个数据库行使用标准 Java 属性(如一个POJO 或 JavaBean )。

GreenDao 的使用

1,添加依赖

```java
//数据库依赖
api 'org.greenrobot:greendao-generator:3.2.0'
api 'org.greenrobot:greendao:3.2.0'
```

 

```java
/**
 * 数据库的插件,在要使用的moudle 中添加插件
 */
apply plugin: 'org.greenrobot.greendao'
```

 

```java
dependencies {
    //noinspection GradleDependency
    classpath 'com.android.tools.build:gradle:3.2.0'
  
    //数据库 在项目的buld.gradle中添加
    classpath 'org.greenrobot:greendao-gradle-plugin:3.2.1'
}
```

 2，创建 存储对象的实体类

添加字段后，只需要在类名上面添加注解 @Entity() 注解 编译就可以生成相应代码，

```java
@Entity(nameInDb = "user_profile")
public class UserProfile {
    @Id
    private String name = null;
    private String avatar = null;
    private String gender  = null;
    private String address = null;
    @Generated(hash = 1032960354)
    public UserProfile(String name, String avatar, String gender, String address) {
        this.name = name;
        this.avatar = avatar;
        this.gender = gender;
        this.address = address;
    }
    @Generated(hash = 968487393)
    public UserProfile() {
    }
    .........
```

```java
@Entity(nameInDb = "person")
public class Person {
    String name;
    int age;
    @Generated(hash = 1687962320)
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    @Generated(hash = 1024547259)
    public Person() {
    }
    public String getName() {
        return this.name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return this.age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```

注解里面的值 为数据库的名称

字段创建完成后，rebuild 一下就会生成对应的代码

2，初始化GreenDao

在Application 中初始化 GreenDao ，采用单例的方式来初始化，

```java
public class ReleaseOpenHelper extends DaoMaster.OpenHelper {

    public ReleaseOpenHelper(Context context, String name) {
        super(context, name);
    }
}
```

```java
public class DatabaseManager {

    private DaoSession mDaoSession = null;
    private DatabaseManager(){}

    public DatabaseManager init(Context context){
        initDao(context);
        return this;
    }

    /**
     * 内部类 的单例模式
     */
    private static final class Holder{
        private static final DatabaseManager INSTANCE = new DatabaseManager();
    }

    /**
     * @return 返回实例
     */
    public static DatabaseManager getInstance(){
        return Holder.INSTANCE;
    }

    /**
     * 初始化，并创建表
     * @param context
     */
    private void initDao(Context context){
        //自定义的类：ReleaseOpenHelper 继承自 DaoMaster.OpenHelper
        //通过 DaoMaster的内部类 可以得到 SQLiteOpenHelper 的对象
        // 第二个参数为 数据库的名称
        final ReleaseOpenHelper helper = new ReleaseOpenHelper(context,"fast_ec.db");
        //获取可读写 数据库
        final Database db =helper.getWritableDb();
        //拿到 DaoSession 对象
        mDaoSession = new DaoMaster(db).newSession();
    }
    public final DaoSession getDao(){
        return mDaoSession;
    }
}
```

3，使用

```java
public class SignHandler {

    private static UserProfileDao getDao(){
        DaoSession dao = DatabaseManager.getInstance().getDao();
        //返回操作的表
        return dao.getUserProfileDao();
    }

    /**
     * @param userProfile 用户注册的数据
     * @param signListener
     */
    public static void onSignUp(UserProfile userProfile,ISignListener signListener){
    	//获取 要操作的表，然后插入数据
        getDao().insertOrReplace(userProfile);       
    }
 }
```