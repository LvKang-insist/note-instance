### 使用 convert 进行 json 和 map 的互相转换

导包

```dart
import 'dart:convert';
```

json 转 map

```dart
const json =
    '{"data":{"isforce":false,"newVersion":true,"download":"https://baidu.com","size":"26"},"message":"SUCCESS","result":1}';
Map<String, dynamic> map = jsonDecode(json);
print(map["data"]);
```

map 转 json

```dart
String str = jsonEncode(map);
print(str);
```

如果不是数据类型特别复杂，推荐使用这种方式进行解析数据，并且这种方式不需要外部的依赖和其他配置，使用较为方便



### Dart 实体类格式

```dart
class UpdateBean {
  UpdateBean({
    this.isforce,
    this.newVersion,
    this.download,
    this.size,
  });

  bool isforce;
  bool newVersion;
  String download;
  String size;

  factory UpdateBean.fromJson(Map<String, dynamic> json) => UpdateBean(
    isforce: json["isforce"],
    newVersion: json["newVersion"],
    download: json["download"],
    size: json["size"],
  );

  Map<String, dynamic> toJson() => {
    "isforce": isforce,
    "newVersion": newVersion,
    "download": download,
    "size": size,
  };
}
```

上面就是一个标准的 dart 实体类

### Dart 三种实体类的创建方式

1. 手写实体类

   根据 json 的字段逐个进行解析从而创建实体类，这种方式比较麻烦，需要手动进行编写代码，出错率较高，所以一般不推荐使用

2. 使用第三方的网页工具

   对于一些比较复杂的 json 串，可以使用一些网页工具来进行转换

   现在一写网页上都可以进行解析转换，直接将 json 继续成一个类，直接复制解析后的内容即可，下面推荐两个网站：

   https://app.quicktype.io/

   https://javiercbk.github.io/json_to_dart/

3. 使用第三方插件或者库

   使用三方库： [json_serializable](https://pub.flutter-io.cn/packages/json_serializable) 使用方式：https://blog.csdn.net/xudailong_blog/article/details/95168949

   使用插件：FlutterJsonBeanFactory

   <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210507111210.png" alt="image-20210507111210129" style="zoom:50%;" />

上面几种方案的对比

| 方案                   | 特点                                                         | 适合场景             |
| ---------------------- | ------------------------------------------------------------ | -------------------- |
| 手写                   | 耗时，很容易出现错误                                         | 小项目和json串比较小 |
| 网页自动生成           | 快速，简单，易操作                                           | 任何类型都可使用     |
| json_serializable      | 需要手动定义字段，易维护                                     | 中大型项目           |
| FlutterJsonBeanFactory | 简单，高效，并且内部有容错判断，不用重复导包等，但是生成的类较多，看着稍微有点烦 | 任何类型都可使用     |

