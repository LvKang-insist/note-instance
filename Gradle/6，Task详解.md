#### 		1，Task 的创建

```java
//直接通过 task 函数创建
task("helloTask") {
    println '--------- i am helloTask'
}

//通过 TaskContainer 去创建 Task
this.tasks.create(name: "helloTask2") {
    println '--------- i am helloTask2'
}
```

​		以上两种方式创建的 Task 没有任何区别，

​		this.tasks 返回的就是 TaskContainer 类，Task 的容器类，上面两种方式创建的 Task 都会被添加当前 project 的 TaskContainer 类中，TaskContainer 是用来管理当前 project 中所有的 Task。既然是用来管理的，他肯定可以对 Taks 进行一些操作，如下：

```java
public interface TaskContainer extends TaskCollection<Task>, PolymorphicDomainObjectContainer<Task> {
	
	//在指定路径下查找 task ，没有返回 null
	 @Nullable
    Task findByPath(String path);
    //同上，没有抛出异常
    Task getByPath(String path) throws UnknownTaskException;
    
    //非常多的创建方法
    Task create(Map<String, ?> options) throws InvalidUserDataException;
    Task create(Map<String, ?> options, Closure configureClosure) throws InvalidUserDataException;
    Task create(String name, Closure configureClosure) throws InvalidUserDataException;
    Task create(String name) throws InvalidUserDataException;

}
```

​		无论使用那种方式创建 Task ，最终 TaskContainer 会将所有的 Task 在配置阶段构成一个有向无环图。通过这个图 gradle 就能知道 task 的类别和执行顺序。

#### 2，Task配置

​		可以在创建的时候直接配置组合描述信息，如下：

```java
task helloTask(group: '345', description: 'Task Study') {
    println '--------- i am helloTask'
}
```

​		也可以在闭包中进行设置描述信息：

```java
this.tasks.create(name: "helloTask2") {
    setGroup("345")
    setDescription("Task Study")
    println '--------- i am helloTask2'
}
```

​		第一种看起来比较简洁，指定 group 后 ，就会将 task 放在对应的组中。description 就相当于是注释。指定分组后，点击 as 右上角的 Gradle ，打开指定 task 的 module。就会显示当前 project 的所有 task。如果指定了分组，则就会有对应的文件夹，如果没有就会放在 other 中，如下：

![1576554059473](6%EF%BC%8CTask%E8%AF%A6%E8%A7%A3.assets/1576554059473.png)

​		当然 task 不会只能配置上面两种信息。打开 Task 类，如下：

```java
public interface Task extends Comparable<Task>, ExtensionAware {
    //名字
    String TASK_NAME = "name";
	//描述
    String TASK_DESCRIPTION = "description";
	//组
    String TASK_GROUP = "group";
	//类型
    String TASK_TYPE = "type";
	//指定当前 task 依赖于那些 task
    String TASK_DEPENDS_ON = "dependsOn";
	//重写 task
    String TASK_OVERWRITE = "overwrite";
	//配置要执行的逻辑
    String TASK_ACTION = "action";
}
```

#### 3，让 Task 执行在 **执行阶段**

​		首先我们有运行上面的例子 gradlew helloTask ，如下：

```java
--------- i am helloTask
--------- i am helloTask2
```

​		这两个都执行了。为啥呢？他们是在 配置阶段执行的，并不是执行阶段，所以他们都会被执行。下面我们就看一下执行阶段。

​		执行阶段是 gradle 生命周期的第三阶段。也只有 Task 才能在第三阶段中执行，要将 Task 执行在 执行阶段，有两个方法，分别是 doFirst 和 doLast ，通过 doFirst 可以为已经有的 task 之前添加相应的逻辑，doLast 则是之后了，下面看一下简单的使用：

```java
task helloTask(group: '345', description: 'Task Study') {
    // doFirst 执行在 执行阶段
    doFirst {
        println '--------- i am helloTask --1'
    }

    //可执行多次
    doFirst {
        println '--------- i am helloTask --2'
    }
    doLast{
        println "last"
    }
}
helloTask.doFirst {
    //可定义在外面
    println '--------- i am helloTask --3'
}
```

打印结果：

```java
> Task :app:helloTask
--------- i am helloTask --3
--------- i am helloTask --2
--------- i am helloTask --1
last
```

在配置阶段执行的时候是不会打印 Task :app:helloTask 这句话的。这里的执行顺序是从外部的开始执行，最后执行的是 doLast 。

**小例子：计算 Build 中 task 的执行时长**

```java
// 计算 build 计算时长
def startBuildTime, endBuildTime
this.afterEvaluate {
    Project p ->
        def preBuildTask = p.tasks.getByName('preBuild')
        preBuildTask.doFirst {
            startBuildTime = System.currentTimeMillis()
            println "this start time is :" + startBuildTime
        }
        def buildTask = p.tasks.getByName('build')
        buildTask.doLast {
            endBuildTime = System.currentTimeMillis()
            println "this build time is :${endBuildTime - startBuildTime}"
        }
}
```

使用命令 gradlew build 开始 build

执行结果：

```java
> Task :app:preBuild
this start time is :1576560417687
> Task :app:build
this build time is :28783
```

其中 this.afterEvaluate 是生命周期中的方法，绘制配置阶段执行所有的 task 配置完成后执行，接着就是获取 preBuild task，他是第一个开始执行的 task，不信你可以看一下控制台的log。在 preBuild 的 doFirst 中获取开始时间，在 build task 完成后获取结束时间即可。

#### 4，Task 执行顺序

​		看上面的小例子：我们执行的命令是 gradlew build ,那为啥先执行 preBuild 呢！这就和 Task 的执行顺序有关了。

![1576561406733](6%EF%BC%8CTask%E8%AF%A6%E8%A7%A3.assets/1576561406733.png)

- 强依赖方式

  为 task 指定多个依赖性的 task。这样当前的 task 必须等依赖的 task 执行完才可以执行

  ```java
  task TaskX {
      doLast {
          println 'taskX'
      }
  }
  task TaskY {
      doLast {
          println 'taskY'
      }
  }
  
  //依赖 taskX ，taskY
  task TaskZ(dependsOn: [TaskX, TaskY]) {
      doLast {
          println 'taskZ'
      }
  }
  // TaskZ.denpendsOn(TaskX, TaskY) 这种也可以
  ```

  执行结果：

  ```java
  > Task :app:TaskX
  taskX
  
  > Task :app:TaskY
  taskY
  
  > Task :app:TaskZ
  taskZ
  ```

  如果在最开始不知道要依赖那些 task，可以通过下面这种方式：

  ```java
  task TaskX {
      doLast {
          println 'taskX'
      }
  }
  task TaskY {
      doLast {
          println 'taskY'
      }
  }
  
  //依赖 taskX ，taskY
  task Test {
      dependsOn this.tasks.findAll {
          task ->
              //测试此字符串是否以指定的前缀开始。
              return task.name.startsWith('Task')
      }
      doLast {
          println 'Test'
      }
  }
  ```

  结果如下：

  ```java
  > Task :app:TaskX
  taskX
  
  > Task :app:TaskY
  taskY
  
  > Task :app:Test
  Test
  ```

  

- 通过 api 来指定顺序

  ```groovy
  //执行顺序指定
  task taskX {
      doLast {
          println 'taskX'
      }
  }
  task taskY {
      //指定执行在 TaskX 之后
      mustRunAfter taskX
      doLast {
          println 'taskY'
      }
  }
  task taskZ {
      //指定执行在 TaskY 之后
      mustRunAfter taskY
      doLast {
          println 'taskZ'
      }
  }
  ```

  输入命令：gradlew taskX taskZ TASKZ 	结果如下：

  ```java
  > Task :app:taskX
  taskX
  
  > Task :app:taskY
  taskY
  
  > Task :app:taskZ
  taskZ
  ```

  

最后看一个例子：

```java
ext {
    versionName = '1.0.1'
    versionCode = 1
    versionInfo = '第1个版本'
    destFile = file('releases.json')
    if (destFile != null && !destFile.exists()) {
        destFile.createNewFile()
    }
}
```

上面的 ext 不会陌生吧，他是扩展属性。内部定义了一写属性，创建了一个文件 releases.json

接着定义一个类，来保存这些属性：

```java
class VersionMsg {
    String versionName
    String versionCode
    String versionInfo
}
```

然后创建一个 Task。用来给文件中写入信息：

```java
task writeTask {
    //为 task 指定输入
    inputs.property('versionName', this.versionName)
    inputs.property('versionCode', this.versionCode)
    inputs.property('versionInfo', this.versionInfo)
    //为 task 指定输出
    outputs.file this.destFile
    doLast {
        //获取输入让的信息，返回 map
        def map = inputs.getProperties()
        File file = outputs.getFiles().getSingleFile()
        def versionMsg = new VersionMsg(map)
        def json = JsonOutput.toJson(versionMsg)
        if (file.text != null && file.text.size() >= 0) {
            def lines = file.readLines()
            def lengths = lines.size() //一共多少行
            file.withWriter {
                writer ->
                    if (lengths == 0) {
                        writer.append("[")
                        //将json串格式化后写入
                        writer.append("${JsonOutput.prettyPrint(json)}\n")
                    } else {
                        for (int i = 0; i < lines.size(); i++) {
                            if (i == lengths - 1) {
                                writer.append(",")
                                break
                            }
                            writer.append(lines[i] + "\n")
                        }
                        writer.append("${JsonOutput.prettyPrint(json)}\n")
                    }
                    writer.append("]")
            }
        }
    }
}
```

inputs 和 outputs 是 Task 的属性，inputs 可以是任意类型的对象，而 outputs只能是文件(或文件夹)

上面这段代码首先是指定了输入和输出，然后获取输入的文件，并传入到 VersionMsg类中，接着将这个类转为 json 串。最后将 json 写入到文件中。

注意要使用 JsonOutput ，必须先导包，否则不能使用，如下：

```java
import groovy.json.JsonOutput
```

最终执行 gradlew writeTask 命令，执行前请修改信息，写入的文件如下：

![1576647952023](6%EF%BC%8CTask%E8%AF%A6%E8%A7%A3.assets/1576647952023.png)

这就是输出的版本信息，其中 contentHash 和 originalClassName 是自动生成的。

在这个过程中也遇到了一个问题：在执行 Task 的时候，如果 Task 内涉及的内容没有发生任何变化，那么这个 task 不会执行。如果上面的执行完第三次后没有修改任何地方，在继续执行 Task 则不会执行。但是只要有任何修改，task 都会执行：如将 文件中的 json 传格式化一下，修改一下等，再次执行命令，则 task 就会执行。

------

到这里就完了，但是还是有些疑惑，在学习的过程中，看到可以使用 输入输出来指定执行的顺序，但是我却没弄出来。还有就是指定在 build Task 后执行，但是定义的 task 没有 execute() 方法，一直报错，无法在代码中使其运行。

希望有解决过的同学讲一哈，多谢