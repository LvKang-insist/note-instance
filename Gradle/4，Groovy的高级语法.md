### json 操作

- 将实体对象转换Wie Json

  ```groovy
  def list = [new Person(name: "张三", age: 20),
              new Person(name: "李四", age: 22),
              new Person(name: "王五", age: 25)];
  def json = JsonOutput.toJson(list)
  println json //输出 json 串
  println JsonOutput.prettyPrint(json) //按照格式输出
  ```

  

- 将 json 串转为对象

  ```java
  List<Person> newJson = new JsonSlurper().parseText(json) as List<Person>
  println newJson[0].name + "----" + newJson[0].age
  ```

  

### XML 

​		xml 对于我来说没用过，所有就。。。

### 文件操作

- 遍历文件

  ```groovy
  def file = new File("../../GroovySpecifcation.iml")
  //遍历文件
  file.eachLine {
      println it
  }
  ```

- 读取文件

  ```java
  def result = file.getText() //获取文件中内容
  println result
  
  def result = file.readLines()//将每一行的数据放在列表中
  println result
  ```

- 获取文件中一部分内容

  ```java
  def reader = file.withReader { //获取文件中的一部分内容
      read ->
          char[] buffer = new char[100];
          read.read(buffer)
          return buffer
  }
  println reader
  ```

- copy 一个文件

  ```java
  static def copy(String patch, String newFile) {
      //创建目标文件
      def file = new File(newFile)
      if (!file.exists()) {
          file.createNewFile()
      }
  
      //copy
      new File(patch).withReader {
          //闭包参数
          read ->
              //将文件读取到列表中
              def lines = read.readLines()
              file.withWriter {
                  write ->
                      //遍历列表，将数据写入到目文件中
                      lines.each {
                          line ->
                              write.append(line + "\n")
                      }
              }
      }
      return true
  }
  
  copy("../../GroovySpecifcation.iml",
          "../../GroovySpecifcation2.iml")
  ```

- 读写对象

  ```groovy
  //保存对象
  static def saveObject(Object o, String filePath) {
      //创建目标文件
      def file = new File(filePath)
      if (!file.exists()) {
          file.createNewFile()
      }
      file.withObjectOutputStream {
          //闭包参数
          out ->
              out.writeObject(o)//保存对象到指定文件中
      }
  }
  
  //读取对象
  static def readObject(String patch) {
      File f = new File(patch)
  
      f.withObjectInputStream {
          read ->
              def obj = read.readObject() //读取对象
              Person p = obj as Person  //转型
              println p.name + "---" + p.age
      }
  }
  
  saveObject(new Person(name: "张三", age: 20), "../../person")
  readObject("../../person")
  ```

  注意：这里的 Person 对象需要被序列化

  ### 总结

  到这里基础的就学的差不多了。感觉还是挺简单的，因为是基于 java 的，所以学起来还是很友好。

   **groovy 和 java 的区别**

  - 写法上：非常随意，没有 java 那么多的限制，例如 不用写逗号，return 可写 可不写，可以直接当做脚本来写，main 方法 和 class 都不需要。
  - 功能上：功能更全面吧，在 java 的基础上有添加了很多方法，并且使用起来非常简单，特别是闭包的使用，简直了。。还有强大的元编程，可以再运行期间动态的生成字段，方法等，非常强大。并且兼容 java 的所有功能
  - 作用上：可以写 应用，也可以写脚本，但大多时候是用来写脚本的。

  **重要的地方**

  - 强定义和弱定义，变量，字符串，循环等基本的语法
  - 数据结构，列表，映射，范围等。注意列表默认是 ArrayList ，映射LinkedHashMap ，范围则是继承自列表的。需要需要改变可以通过 as 来修改，如 def list = [1,4,6] as LinkedList。
  - 类，方法，面向对象，这些基本和 java 差不多，只不过 定义方法时 def 代表的返回值就是 Object 了。而 类默认继承的是 GroovyObject，而不是 Object。
  - 闭包，这个是一个比较重要的东西，说一下笔者的理解：可以把它看成一个代码块，这个代码块有参数，有返回值，而且还可以作为参数传递。groovy 中的很多方法都需要传入一个闭包，这个闭包的功能由我们自己来确定，groovy 在这个方法的内部就会调用闭包，然后闭包就会执行。所以在使用 groovy 一写需要传入闭包的方法时，不妨先看一下方法内部的实现，就算看不懂，你也需要看一下方法内部在调用闭包是有没有传入参数，传了几个，是什么类型！！！ 
  - json格式，文件的处理，感觉实在 java 的基础上进行封装，整体上感觉方便多了，再也不是要 try catch 去关闭流了。。。

  - 元编程，在运行的时候可以动态的注入字符，方法，静态方法，默认是对当前的文件生效，但是可以修改配置，使其全局生效。