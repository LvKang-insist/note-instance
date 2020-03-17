Room 数据库

Room 是 Google 为了简化旧式的 SQLite 操作专门提供的

- 拥有 SQLite 的所有操作功能
- 使用简单（类似于 Retrofit库），通过注解的方式实现相关功能。编译时自动生成实现了 impl
- LiveData，LifeCycle ，Paging 天然融合。

### 1，数据库的创建

```kotlin
@Database(
    entities = [Cache::class], version = 1, exportSchema = true
)
abstract class CacheDatabase : RoomDatabase() {
    companion object {
        init {
            //创建一个内存数据库，这种数据库只存在于内存中，进程被杀后，数据随之丢失
//            Room.inMemoryDatabaseBuilder()
            var database = Room.databaseBuilder(
                AppGlobals.getApplication(), CacheDatabase::class.java, "ppjoke_cache"
            )
                //是否允许在主线程进行查询
                .allowMainThreadQueries()
                //打开和创建的回调
//                .addCallback()
                //设置查询时的线程池
//                .setQueryExecutor()
                //设置数据库工厂
//                .openHelperFactory()
                //room 的日志模式
//                .setJournalMode()
                //数据库升级异常后的回滚
//                .fallbackToDestructiveMigration()
                //数据库升级异常后根据指定版本进行回滚
//                .fallbackToDestructiveMigrationFrom()
                //向数据库添加迁移，每次迁移都要有开始和最后版本，Room 将将迁移到最新版本
                //如果没有迁移对象，则数据库会重新创建
//                .addMigrations(CacheDatabase.sMigration)
                .build()
        }

        /**
         * 数据库迁移对象，可以对数据库进行必要的更改
         */
        private val sMigration = object : Migration(1, 3) {
            override fun migrate(database: SupportSQLiteDatabase) {
                database.execSQL("alter table teacher rename to student")
                database.execSQL("alter table teacher add column teacher_age INTEGER NOT NULL default 0 ")
            }
        }

    }

}
```

​	注解中第一个是实体类的class，第二个是版本号，第三个为是否输出文件，如果设置为 true，则需要设置文件的位置，如下：

```kotlin
 defaultConfig {
 		.......
        //Room 数据库生成的文件保存在哪个位置上面：当前 module 下 schemas文件中
        javaCompileOptions{
            annotationProcessorOptions{
                arguments=["room.schemaLocation":"$projectDir/schemas".toString()]
            }
        }
    }
```

### 2，注解的使用

- #### @ColumnInfo

  ```kotlin
  class Cache : Serializable {
      var key: String? = null
      
      //以这个字段的名字为列的名字，也可以自定义名字如 @ColumnInfo(name="_data")
      //默认可以不写
      @ColumnInfo
      var data: Array<Byte>? = null
  }
  ```

- #### @Dao

  将类标记为数据访问对象，可以对数据库进行查询等，被标记的类必须是接口或者抽象类。ReBuild 后机会生成对应的类，来正真实现增删改查 

  ```kotlin
  @Dao
  interface CacheDao {
      //插入一条数据，如果有冲突，按照指定的方式执行
      //有多种执行的方式
      @Insert(onConflict = OnConflictStrategy.REPLACE)
      fun save(cache: Cache): Long
  
      //查询，cache 表中 key 的列
      @Query("select * from cache where `key`=:key")
      fun getCache(key: String)
  
      //删除
      @Delete
      fun delete(cache: Cache)
  
      //更新一跳数据
      @Update(onConflict = OnConflictStrategy.REPLACE)
      fun update(cache: Cache)
  }
  ```

- #### @Database

  将一个类标记为 RoomDatabase(数据库)

- #### @Delete

  将方法标注为删除方法，可参数集合，实体类等

- #### @Embedded

  嵌套对象

  ```kotlin
  class Cache : Serializable {
      var key: String? = null
  
      @ColumnInfo
      var data: Array<Byte>? = null
  
  
      //让 user 中的字段出现在 Cache 表当中
      @Embedded
      var user: User? = null
  }
  ```

- #### @Entity ，@PrimaryKey

  将类标记为实体类，这个类在数据库中有一个表

  ```kotlin
  @Entity(tableName = "cache")
  class Cache : Serializable {
  
      //主键约束，保证 key 的唯一性
      //如果是 int 类型，可以使用 autoGenerate 参数来自动生成主键的值
      @PrimaryKey
      var key: String? = null
  
      @ColumnInfo
      var data: Array<Byte>? = null
  }
  ```

- #### @ForeignKey

  外键，

  ```kotlin
  //ForeignKey：1，相关联表的名称，2，相关联表中的列名，3，当前表中的列名，4，删除，5，更新
  @Entity(
      tableName = "cache", foreignKeys = [ForeignKey(
          entity = User::class,
          parentColumns = arrayOf("name"),
          childColumns = arrayOf("key"),
          onDelete = ForeignKey.RESTRICT,
          onUpdate = ForeignKey.SET_DEFAULT
      )]
  )
  class Cache : Serializable {
  
      //主键约束，保证 key 的唯一性
      @PrimaryKey
      var key: String? = null
  
      @ColumnInfo
      var data: Array<Byte>? = null
  
  }
  ```

- #### @Ignore

  忽略，如果将这个注解添加到字段上面，在映射的时候数据库中则不会创建这个字段的列

- #### Index

  在实体上声明索引。

  ```kotlin
  @Entity(
      tableName = "cache", 
      indices = [Index(value = arrayOf("key", "id"))]
  )
  ```

- #### @Insert

  插入数据

- #### @OnConflictStrategy

  各种 Dao 发生冲突时的解决办法,冲突一般存在相同的主键等。

  ```
  /**
   * 替换旧数据并继续事务
   */
  int REPLACE = 1;
  /**
   * 回滚
   */
  int ROLLBACK = 2;
  /**
   * 中止
   */
  int ABORT = 3;
  /**
   * 失败
   */
  int FAIL = 4;
  /**
   * 忽略
   */
  int IGNORE = 5;
  ```

- #### @Query 

  查询操作

  ```key
  //查询，cache 表中 key 的列
  @Query("select * from cache where `key`=:key")
  fun getCache(key: String)
  ```

- #### @RawQuery

  需要动态的传入 Sql 语句

- #### @Relation

  关联查询

  ```kotlin
  class Cache : Serializable {
  
      //主键约束，保证 key 的唯一性
      @PrimaryKey
      var key: String? = null
  
      @ColumnInfo
      var data: Array<Byte>? = null
  
      //查询 User 数据
      @Relation(entity = User::class,parentColumn = "id",entityColumn = "id",projection = [])
      var user: User? = null
  }
  ```

- #### @Transaction

  如果一个方法标注了此注解，则每一个 sql 语句就会被当成一个事务来提交

- #### @TypeConverter 

  将方法标记为类型转换器。一个类可以有它需要的任意多个@TypeConverter方法。

  ```kotlin
  class DataConverter {
  	//进行标记
      @TypeConverter
      fun data2Long(date: Date): Long = date.time
  
      @TypeConverter
      fun long2Date(long: Long): Date = Date(long)
  }
  ```

  ```kotlin
  class Cache : Serializable {
  
      //主键约束，保证 key 的唯一性
      @PrimaryKey
      var key: String? = null
  
      @ColumnInfo
      var data: Array<Byte>? = null
  
  	//在 保存 mDate 时就会转成 Long，在读取时就会转成 Date
      @TypeConverters(value = [DataConverter::class])
      val mDate: Date? = null
  }
  ```

- #### @TypeConverters

  如上，标记在字段上面的。指定使用的类型转换器，可标记在数据库，字段，类，方法等。

- #### @upDate

  更新操作，有冲突，需要使用上面的注解进行标注出现冲突后的操作

