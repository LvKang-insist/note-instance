## PHP学习

### GET请求

```php

<?php
//判断当前请求是否为空数组
if(empty($_GET)){
    //为null
}else{
    //不为null
    $name=$_GET['name'];
    $age=$_GET['age'];
    $graduateId=$_GET['graduateId'];
    //打印出GET请求中传递的参数
    echo $name.'今年'.$age.'毕业于'.$graduateId;
}
其请求方式如下：
格式:index.php?name=jack&age=34&$graduateId=新东方 （注意：index.php？key=value&key=value。 name=jack&age=34&$graduateId=新东方 叫做查询字符串）
参数名与参数值之间没有空格
参数值不需要使用单双引号包括

其GET方式通过URL传递参数，受限于URL的长度，并且GET方式的参数信息会在URL中明文显示。

```

### POST请求

```php
//以表单的形式进行提交
<?php
//判断当前请求是否为空数组
if(empty($_POST)){
    //为null
}else{
    //不为null
    $name=$_POST['name'];
    $age=$_POST['age'];
    $graduateId=$_POST['graduateId'];
    //打印出GET请求中传递的参数
    echo $name.'今年'.$age.'毕业于'.$graduateId;
}

/*
以JSON形式进行提交
这里说明一下，当android端直接以json字符串格式往php脚本的服务器上传数据时，一般的$_POST是接收不到的
因为PHP默认只识别application/x-www.form-urlencoded的标准数据类型，所以对text/xml,或者soap或者octet-stream之
类的内容无法解析
所以这时就用到了file_get_contents("php://input")获取android端传来的json字符串，然后json_decode($json,true)将
json字符串转为数组
*/

$json = file_get_contents("php://input")；
$data =json_decode($json, true);
 $name=$data['name'];
    $age=$data['age'];
    $graduateId=$data['graduateId'];
echo $name.'今年'.$age.'毕业于'.$graduateId;
```

```php
<?php

if (!empty($_POST)){
    $name=$_POST['name'];
    $age=$_POST['age'];
    $graduateId=$_POST['graduateId'];
    echo $name.'今年'.$age.'毕业于'.$graduateId;
}else{
    $json = file_get_contents("php://input");
    $data = json_decode($json, true);
    $name=$data['name'];
    $age=$data['age'];
    $graduateId=$data['graduateId'];
    echo $name.'今年'.$age.'毕业于'.$graduateId;
}
```



使用Postman进行表单提交

![1561689308554](C:\Users\respond\AppData\Roaming\Typora\typora-user-images\1561689308554.png)

使用Postman进行json格式提交

![1561689350064](C:\Users\respond\AppData\Roaming\Typora\typora-user-images\1561689350064.png)

### 数据库操作

数据库的操作为三种操作：

​	一种是mysqli面向对象的操作，一种是mysqli面向过程的操作，一种是PDO操作，

​	MySQLi和PDO有它们自己的优势：

​	PDO应用在12种不同数据库中，MySQLi只用于MySQL数据库。

​	所以，如果你的项目需要在多种数据库中切换，建议使用PDO，这样你只需要修改连接字符串和部分查询语句即可。使用MySQLi，如果不同数据库，你需要重新编写所有代码，包括查询。

#### 连接数据库

```php
<?php
header('content-type:text/html;charset-utf-8');
function connect()
{   
    //IP
    $servername = "localhost";
    //用户名
    $username = "root";
    //密码
    $password = "";
    //数据库名
    $dbname = 'demo';
    // 创建连接
    $conn = new mysqli($servername, $username, $password, $dbname);

    // 检测连接
    if (!$conn) {
        die("Connection failed: " . $conn->connect_error());
    }
    //设置数据库格式
    $conn->query("set names utf8");
    return $conn;
}
```

#### 插入数据

```php
<?php /** @noinspection ALL */
header('content-type:text/html;charset-utf-8');
require('./connect.php');
//插入成功返回的JSON串
$result_ok = array(
    'result' => 'OK',
    'msg' => '插入成功'
);
//返回一个JSON串
$json_encode = json_encode($result_ok, JSON_UNESCAPED_UNICODE);

//插入失败返回的JSON串
$result_error = array(
    'result' => 'Error',
    'msg' => '插入失败'
);
$json_error = json_encode($result_error, JSON_UNESCAPED_UNICODE);

//连接数据库，获取数据库对象
$conn = connect();

//插入语句
$sql = "INSERT INTO `search_custom_mall` (`id`, `name`, `password`, `time`, `login`) VALUES (NULL, 'qwe1', 'qwe1', 'qwe1', 'qwe1')";
$result = $conn->query($sql);

if ($result) {
    echo $json_ok;
} else {
    echo $json_error;
}
//关闭数据库连接
$conn->close();

```

#### 查询数据

```php
<?php
header('content-type:text/html;charset-utf-8');
require ('./connect.php');
//连接数据库,获取mysqli对象
$conn=connect();
//创建查询语句
$sql="select * from search_custom_mall";

//进行查询
$result=$conn->query($sql);
//返回结果集中的行数是否大于0
if ($result->num_rows>0){
    //将结合集放入到关联数组并循环输出
    $emp_info=array();
    while($row=$result->fetch_assoc()){
        //将取出的每条结果集装入emp_info数组中
        $emp_info[]=$row;
    }
    /*
    *json_encode()
    *对于索引数据的内容会以[]包裹内容，对于关联数据中的内容会以{}进行处理
    */
    echo json_encode($emp_info,JSON_UNESCAPED_UNICODE);
}else{
    echo '0 结果';
}
$conn->close();
```

#### 删除数据

```
<?php
require ('./connect.php');
//删除成功返回的JSON串
$result_ok = array(
    'result' => 'OK',
    'msg' => '删除成功'
);
//返回一个JSON串
$json_encode = json_encode($result_ok, JSON_UNESCAPED_UNICODE);

//删除失败返回的JSON串
$result_error = array(
    'result' => 'Error',
    'msg' => '删除失败'
);
$json_error = json_encode($result_error, JSON_UNESCAPED_UNICODE);
//数据进行连接，并且得到数据库对象
$conn = connect();
$sql="DELETE FROM `search_custom_mall` WHERE `search_custom_mall`.`name` = '321'";
$result = $conn->query($sql);
if ($result){
    echo $json_encode;
}else{
    echo $json_error;
}
```

#### 修改数据

```php
<?php
require ('./connect.php');

//修改成功返回的JSON串
$result_ok = array(
    'result' => 'OK',
    'msg' => '插入成功'
);
//返回一个JSON串
$json_encode = json_encode($result_ok, JSON_UNESCAPED_UNICODE);

//修改失败返回的JSON串
$result_error = array(
    'result' => 'Error',
    'msg' => '插入失败'
);
$json_error = json_encode($result_ok, JSON_UNESCAPED_UNICODE);

//连接数据库，获取数据库对象
$conn=connect();
//设置修改语句
$sql="UPDATE `search_custom_mall` SET `name` = 'nnnnn' WHERE `search_custom_mall`.`name` = 'qwe';";
$result = $conn->query($sql);
if ($result){
    echo $json_encode;
}else{
    echo $json_error;
}
```

### 对于以上所述进行整理Demo

#### 使用GET请求进行查询

PHP端的写法（android用get请求发送json串到服务端）

```php
<?php
header('content-type:text/html;charset-utf-8');
//如果GET请求参数为null，则返回{"result":"error"}
$error = json_encode(array(
    "result" => "error"
), JSON_UNESCAPED_UNICODE);

@$db_table = $_GET['db_table'] ? $_GET['db_table'] : "";
if ($db_table != "") {
    $servername = "localhost";
    $name = "root";
    $password = "";
    //@为抑制错误
    $conn = null;
    @$conn = new mysqli($servername, $name, $password, $db_table);
    //判断数据库是否连接错误
    if (mysqli_connect_error()) {
        echo $error;
    } else {
        $conn->query("set names utf8");
        $sql = "SELECT * FROM `search_custom_mall`";
        $result = $conn->query($sql);

        $arr = array();
        while ($row = $result->fetch_assoc()) {
            $arr[] = $row;
        }
        $msg = array("result" => "ok", "msg" => $arr);
        echo json_encode($msg, JSON_UNESCAPED_UNICODE);
    }

} else {
    echo $error;
}
```

请求参数未设置或者失败时返回

![1561793098745](C:\Users\respond\AppData\Roaming\Typora\typora-user-images\1561793098745.png)

![1561793110429](C:\Users\respond\AppData\Roaming\Typora\typora-user-images\1561793110429.png)

正确请求时返回的JSON格式

![1561793140698](C:\Users\respond\AppData\Roaming\Typora\typora-user-images\1561793140698.png)

#### 使用POST请求进行查询

```php
<?php

header('content-type:text/html;charset-utf-8');

if (!empty($_POST)){
    $name=$_POST['name'];
    query($name);
}else{
    $json = file_get_contents("php://input");
    $data = json_decode($json, true);
    $name=$data['name'];
    query($name);
}

function query($username){
    //如果GET请求参数为null，则返回{"result":"error"}
    $error = json_encode(array(
        "result" => "error"
    ), JSON_UNESCAPED_UNICODE);

    if ($username != "") {
        $servername = "localhost";
        $name = "root";
        $password = "";
        $db_name="demo";
        //@为抑制错误
        @$conn = new mysqli($servername, $name, $password,$db_name);
        //判断数据库是否连接错误
        if (mysqli_connect_error()) {
            echo $error;
        } else {
            $conn->query("set names utf8");
            $sql = "SELECT * FROM `search_custom_mall` WHERE `search_custom_mall`.`name` = '$username'";
            $result = $conn->query($sql);

            $arr = array();
            while ($row = $result->fetch_assoc()) {
                $arr[] = $row;
            }
            $msg = array("result" => "ok", "msg" => $arr);
            echo json_encode($msg, JSON_UNESCAPED_UNICODE);
        }
    } else {
        echo $error;
    }
}
```

使用POST的表单形式提交

![1561795716045](C:\Users\respond\AppData\Roaming\Typora\typora-user-images\1561795716045.png)

使用POST的JSON串进行提交

![1561795746794](C:\Users\respond\AppData\Roaming\Typora\typora-user-images\1561795746794.png)

### 处理文件上传

#### API

##### $FILES数组介绍

其数组中的error有7个值：

​	0表示上传成功

​	1表示文件大小超过了php.ini中的upload_max_filesize选项限制的值

​	2表示文件大小超过了表单中max_file_size选项指定的值，

​	3表示文件只有部分被上传

​	4表示没有文件被上传

​	6表示找不到临时文件夹

​	7表示写入失败

```PHP
array (size=1)
  'file' => 
    array (size=5)
      'name' => string '1_1.png' (length=7)
      'type' => string 'image/png' (length=9)
      'tmp_name' => string 'D:\wamp\wamp\tmp\php20B2.tmp' (length=28)
      'error' => int 0
      'size' => int 104398
```

##### $move_uploaded_file()介绍

该方法传递两个参数：

​	第一个参数：上传的文件的文件名。

​	第二个参数：上传后保存的文件位置

#### 实现保存多个上传文件

```php
<?php
//查看文件信息
var_dump($_FILES);
$error='';
$path = 'D:\wamp\wamp\www\file';
foreach($_FILES as $key=>$value){
    $file= $_FILES[$key];
    $newfilename = fileupload($file, $path, $error);
    echo $path.'\\'.$newfilename;
    //将上传的文件地址保存至数据库
}

/*
 * @param1 array $file 要上传的文件信息，包含5个元素
 * @param2 string $path 存储位置
 * @param3 string $error 错误信息
 * @param4 array $type=array() MIME类型限定
 * @param5 int $size=2000000 默认为2M
 * @return mixed，成功返回文件名字，失败返回false
 *
 * */
function fileupload($file,$path,&$error,$type=array(),$size=2000000){
    //判断文件本身是否是有效文件
    if (!isset($file['error'])){
        $error='文件无效';
        return false;
    }

    if (!is_dir($path)){
        $error='存储路径无效';
        return false;
    }
    //判断文件是否上传成功
    switch ($file['error']){
        case 1:
        case 2:
            $error='文件超过服务器允许大小';
            return false;
        case 3:
            $error='文件部分上传成功';
            return false;
        case 4:
            $error='用户没有选择上传的文件';
            return false;
        case 6:
        case 7:
            return false;
    }
    //类型判定
    if (!empty($type)&&!in_array($file['type'],$type)){
        $error='当前上传的文件类型不符合';
        return false;
    }

    //大小判定
    if ($file['size']>$size){
        $error='文件大小超过当前允许的大小';
    }

    //文件转存
    $newfilename = getNewName($file['name'],5);
   if ( move_uploaded_file($file['tmp_name'],$path.'\\'. $newfilename)){
       //成功
       return $newfilename;
   }else{
       return "存入失败";
   }
}


//随机生成文件名
function getNewName($filename,$rand){
    $newname=date('YmdHis');
    //将参数数组合并为一个数组
    $old=array_merge(range('a','z'),range('A','Z'),range('0','9'));
    //将数组打乱
    shuffle($old);
    for ($i=0;$i<$rand;$i++){
        $newname.=$old[$i];
    }
    //获取有效文件名
    return $newname.strstr($filename, '.');
}
```

多个文件上传时的格式

![1561805465685](C:\Users\respond\AppData\Roaming\Typora\typora-user-images\1561805465685.png)