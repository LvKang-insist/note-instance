**FastJson是阿里巴巴开源的一个Json处理工具包，包括“序列化”和“反序列化”两部分。fastjson具有极快的性能，超越任其他的Java Json parser。包括自称最快的JackJson，功能强大，完全支持Java Bean、集合、Map、日期、Enum，支持范型，支持自省；无依赖，能够直接运行在Java SE 5.0以上版本；支持Android；开源 (Apache 2.0)。**

常用方法：

**public static final String toJSONString(Object object); // 将object序列化为json字符串** 

**public static final String toJSONString(Object object, boolean prettyFormat); // 将object序列化为带格式的json字符串**  

例子：

#### 将map 转为Json 串

```java
Map<String, String> map = new HashMap<String, String>();
map.put("age", "25");
map.put("name", "张三");
map.put("sex", "男");
String mapJson = JSON.toJSONString(map);
Log.d("TAG","mapJson----:"+mapJson);

mapJson----:{"sex":"男","name":"张三","age":"25"}
```

#### 将List 转成字符串：

```java
Student student1=new Student();
student1.setAge("20");
student1.setName("张三");
student1.setSex("男");
 
Student student2=new Student();
student2.setAge("22");
student2.setName("小华");
student2.setSex("女");
 
List<Student> list=new ArrayList<Student>();
list.add(student1);
list.add(student2);
 
String listJson=JSON.toJSONString(list);
Log.d("TAG","listJson----:"+listJson);

listJson----:[{"age":"20","name":"张三","sex":"男"},{"age":"22","name":"小华","sex":"女"}]
```

#### 将javaBean 转为字符串

**public static final JSONObject parseObject(String text)； // 把JSON文本parse成JSONObject**   

```java 
String Jsonstring="{\"age\":\"20\",\"name\":\"张三\",\"sex\":\"男\"}";
com.alibaba.fastjson.JSONObject object=JSON.parseObject(Jsonstring);
String age=object.getString("age");
String name=object.getString("name");
String sex=object.getString("sex");
 
Log.d("TAG","age----:"+age);
Log.d("TAG","name----:"+name);
Log.d("TAG","sex----:"+sex);

age----:20
name----:张三
sex----:男
```

解析纯 JsonArray 类型数据：

```
String Arraystring="[{\"age\":\"20\",\"name\":\"张三\",\"sex\":\"男\"}," +
                "{\"age\":\"22\",\"name\":\"小华\",\"sex\":\"女\"}]";
com.alibaba.fastjson.JSONArray array=JSON.parseArray(Arraystring);
int length=array.size();
for(int i=0;i<length;i++){
   com.alibaba.fastjson.JSONObject jsonObject=array.getJSONObject(i);
   String ages=jsonObject.getString("age");
   String names=jsonObject.getString("name");
   String sexs=jsonObject.getString("sex");
   Log.d("TAG","ages----:"+ages);
   Log.d("TAG","names----:"+names);
   Log.d("TAG","sexs----:"+sexs);
}
```

