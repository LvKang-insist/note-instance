### HTTP 是什么

- 浏览器地址栏输入地址，打开网页
- Android 中发送网络请求，返回对应内容

HyperText transfetr protocol 超文本传输协议

- 超文本：在电脑中显示的，含有可以指向其他文本的链接文本

### HTTP 的工作方式

​		最直观的方式：地址栏输入地址，然后页面就会显示结果

​		实际上是：在地址栏输入地址，回车后就会向服务器发送请求，然后服务器会进行响应，接着浏览器就会将响应的数据进行渲染出来

![image-20200615170652176](Http%E7%9A%84%E5%8E%9F%E7%90%86%E5%92%8C%E5%B7%A5%E4%BD%9C%E6%9C%BA%E5%88%B6.assets/image-20200615170652176.png)

### URL -> HTTP 报文

- 示例：http://www.baidu.com/s?cl=3 
  - 其中 http 为协议类型，协议类型都有 http，ftp 等等，这些叫做应用层协议
  - //www.baidu.com 为服务器地址
  - /s 及其后面的为 路径 path

上面这个地址看起来是一个地址，但是浏览器拿到后会分为三块进行处理，处理完成后如下所示

- GET /s?cl=3  HTTP/1.1

  Host :www.baidu.com

  

### HTTP 的工作方式

#### 报文格式：Request

![1592226900221](Http的原理和工作机制.assets/1592226900221.png)

**其中第一行为：请求行 ，分别对应三个部分，**

- 第一部分 method ，他可以为 GET ,POST ,PUT 等，代表你的行为
- 第二部分：path ，路径，给服务器看到，定位
- 第三部分：版本

**第二行往下为 headers** 

​	其中左边是键，右边是值

**最后一行为 body**

​	在 get 中是不需要 body 的。而在 post 中则需要使用 body了，body 中是你需要向服务器提交的一些数据

#### 报文格式：Response

![1592227348796](Http的原理和工作机制.assets/1592227348796.png)

**其中第一行为状态行**

- 最开始为 http 的版本
- 中间是 status code，状态码
- 最后是状态信息 status message

**中间的是 headers**

​	左边是键，右边是值

**最后一行为 body**

​	服务器返回的数据

#### 关键内容

- Request method

   请求的方法：如 get，post，put 等

   - GET

      获取资源，不会携带 body

   - POST

      增加或者修改资源，有 body

      ```java
      //请求行
      POST /users  HTTP/1.1
      //headers
      Host:192.168.43.253
      Content-Type:text/html; charset=UTF-8
      Date:Mon, 15 Jun 2020 13:20:08 GMT
      //body
      {
          "name": "456",
          "password": "12345"
      }
      ```

   - PUT

      修改资源，只用来修改，有 body

      和 get 的共同点：他们都是幂等的。意思就是他们的结果是相等的。例如获取资源，获取10 次结果都是一样的，执行多次和执行一次是一样的

   - DELETE

      删除资源，没有body，具有幂等性，删除一次和删除多次没区别

   - HEAD

      和 GET 几乎一样，区别就是服务器不会返回 body。

- Response status code

   对结果做出类型化的描述，如果获取成功，内容未找到等

   - 1xx：临时性消息
   - 2xx：成功
   - 3xx：重定向，会进行二次请求，进行请求后会重新定向，接着就会重新请求请求，这个过程感知不到的，但是可以通过状态码来查看到。301 永久性迁移，302 临时迁移，304 内容没有改变
   - 4xx：客户端错误，如 400参数错误，404 资源未找到等
   - 5xx：服务器错误

- header

   **作用**：

   - HTTP 消息的元数据(metadata)

   - 指的是数据的数据 meta... 都指的是 什么的什么

   -  例子：在游戏闯关的过程中会出现一些奖励关，通过后就会奖励一条命等。这种游戏中的小游戏

   -  元数据：例如在清单文件中配置 activity，我们可以配置name ，theme 等，还有一些数据我们可以自行配置，这些就是 metadata，这就是元数据

   -  HTTP 中的元数据：例如同 http 提交一个用户信息，信息中的 name，age 等都是数据本身，他们并不是元数据。元数据是消息的长度，消息的格式，返回的数据集，有没有压缩等。可以看做为数据的属性

   **比较常见的**

   -  Host：服务器地址，但是他不是用来寻址的。在浏览器将报文拼好之后，在发出请求之前就已经去进行服务器的寻址了，他会拿着域名通过 DNS 去寻找对应的 ip 地址。一个域名可能会对应多个 ip 地址。找到对应的目标服务器地址后就会将报文发给目标服务器。既然都找到服务器了，为啥还要发送 host 呢？因为一个 ip 地址下面可能会有多个服务器存在，他们对外的ip 都是一样的，当寻找到 ip 进行访问的时候就没办法转到具体的主机，这个时候就没办法正确的响应。所以需要传入 host，这个 host 一般传入的是服务器地址+TCP 端口。

   -  Content-Type/Content-Length：Body 的类型/长度(字节)

      - text/html

         ```java
         Content-Type:text/html; charset=UTF-8
         ```

      - application/x-www-form-urlencoded:普通表单,encoded URL 格式

         例如使用 Retrofit 的 post 请求是，需要使用 @FormUrlEncoded ，这种就是表单提交

         而且如果要传文件，还需要使用 @MultPart

      - multpiipart/form-data ; boundary = ********************

         多补发形式，一般用于传输包含二进制内容的多项内容

         使用 Retrofit 就需要使用 @MultPart

         boundary  是一个分界线，用来分界 header 值得各个属性和 body

      - application/json

         json 形式，用于 Web Api 的响应或者 POST/PUT 请求

         body 中是 json 格式

         使用这种方式进行post 请求的时候 body 中的内容可以为 json 格式，好处就是格式比较自由。

      - image/jped	

         图片上传         

   - Location：重新定向的目标 URL

   - User-Agent：用户代理

   - Range/Accept-Range：指定 Body 的内容范围

      如果你的服务器支持你分段取内容/下载 的时候，指定的范围

      例如：从百度上复制一个图片地址到 postMan 中进行求，就会加载出图片，接着你在查看他的 headers，就会找到 Accept-Ranges 这个值如下

      ```
      Accept-Ranges:bytes
      ```

      这表示他是支持分段加载的

      我们在请求这个图片的时候在header 中加上如下属性：

      ```
      Range:bytes=0-15000
      ```

      ![1592235952093](Http的原理和工作机制.assets/1592235952093.png)

      你就会发现这个图片只被加载了一般

      那么他又什么作用呢？1，断点续传，如果下载中断，那么下一次下载的时候只需要指定开始的位置和文件的大小即可。2，分段下载，可以用来多线程下载

   - Cookie/Set-Cookie：发送 Cookie /设置 Cookie

   - Authorization:授权信息

   - Accept：客户端能接受的数据类型

      ```
      Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
      ```

   - Accept-Charset:客户端能接受的字符集，如 utf-8

   - Accept-Encoding：客户端能接收的压缩编码类型，如 gzip

   - Content-Encoding：压缩类型,如 gzip

- body

   body 中一般存放的是数据，如 服务器返回的数据就在 body 中，post 请求是发送的数据也在 body 中

   