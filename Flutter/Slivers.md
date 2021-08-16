### 概述

CustomScrollView：一个滚动的容器，改组件不接受任何 child，但是你可以直接提供 `Slivers` 已创建各种滚动效果，**例如页面中有多个可滑动的列表，如 Appbar， 列表，网格**，等这种就可以直接使用 `SliverAppBar`,`SliverList` 和 `SliverGrid`

Slivers 不是单独指一个组件，而是指的一个系列，所以以 Sliver 开头的组件都是这个系列的，但是他们都只能作用于 `CustomScrollView` 中。

常用到的 Sliver 有，SliverAppbar，SliverList，SliverGrid，SliverToBoxAdapter 等

>由于 CustomScrollView 的子组件只能是 Sliver 系列，如果要将一个普通的组件放在里面，必须使用 `SliverToBoxAdapter` 进行适配才行



### 简单的使用

```dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(widget.title)),
      drawer: Drawer(),
      body: CustomScrollView(
        slivers: [
          SliverAppBar(
            title: Text("SliverAppbar"),
          ),
          SliverGrid(
            gridDelegate:
                SliverGridDelegateWithFixedCrossAxisCount(crossAxisCount: 4),
            delegate: SliverChildBuilderDelegate((context, index) {
              return Container(
                  color: Colors.primaries[index % Colors.primaries.length]);
            }, childCount: 40),
          ),
          SliverList(
              delegate: SliverChildBuilderDelegate((context, index) {
            return Container(
                height: 100,
                color: Colors.primaries[index % Colors.primaries.length]);
          }, childCount: 20))
        ],
      ),
    );
  }
}
```

运行效果如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810135625.gif" alt="345" style="zoom:50%;" />

其实我们仔细一点就会发现，其实 ListView 和 GridView 等组件内部使用的都是 Slivers，

```dart
ListView.builder({
 //......
}) : assert(itemCount == null || itemCount >= 0),
     assert(semanticChildCount == null || semanticChildCount <= itemCount!),
     childrenDelegate = SliverChildBuilderDelegate(
       itemBuilder,
       childCount: itemCount,
       addAutomaticKeepAlives: addAutomaticKeepAlives,
       addRepaintBoundaries: addRepaintBoundaries,
       addSemanticIndexes: addSemanticIndexes,
     ),
     super(
 		//....
     );
```

那为什么要使用 Slivers 呢？最主要的原因就是可以在 slives 中添加多个组件，如在列表的上面和下面添加更多的内容。

并且 slivers 中，**如果存在多个列表的话也是支持动态加载的**，而不是会一次性全部渲染完

___

### 各式各样的 Slivers 组件

#### SliverList

在上面的例子中 SliverList 使用的是 `SliverChildBuilderDelegate` 这个delegate，它可以实现动态加载，当然 SliverList 中也有和 ListView 中一样的非动态加载的delegate，就是`SliverChildListDelegate`

```dart
SliverList(
    delegate: SliverChildListDelegate(
  [
    FlutterLogo(size: 100),
    FlutterLogo(size: 100),
    FlutterLogo(size: 100),
  ],
))
```

一般在列表数量较小并且显示内容确定的情况下可以使用次 `delegate` 。

#####  SliverFixedExtentList

面的子元素中的宽高是动态的，需要手动设置高度，并且这种也不利于性能，所以我们可以使用 `SliverFixedExtentList` 来控制限制子元素的大小：

```dart
SliverFixedExtentList(
    itemExtent: 100,
    delegate: SliverChildListDelegate(
      [
        FlutterLogo(),
        FlutterLogo(),
        FlutterLogo(),
      ],
    ))
```

未限制前：![image-20210810143917248](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810143917.png)，限制后：![image-20210810144017236](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810144017.png)

##### SliverPrototypeExtentList

一般情况下，只要固定了列表中元素的高度，就可以提升不小的性能，但是在实际的项目中，想要固定元素的高度是非常麻烦的，就算是列表中的元素只有一行文字，也有可能会出现问题，例如直接在系统层面修改字体的大小，这也会导致高度的固定导致渲染出来的效果不尽人意。但是有了 SliverPrototypeExtentList 就简单多了。

在 SliverPrototypeExtentList 中，可以通过 prototypeItem 来传入一个原型，这个原型并不会渲染到屏幕上，在运行的过程中，Flutter 会将原型的尺寸计算出来，之后就会把所有的元素尺寸设置成这个原型的尺寸。

```dart
body: DefaultTextStyle(
  style: TextStyle(fontSize: 60, color: Colors.red),
  child: CustomScrollView(
    slivers: [
      SliverPrototypeExtentList(
          prototypeItem: Text(""),
          delegate: SliverChildListDelegate(
            [
              Text("Hello Word"),
              Text("Hello Word"),
              Text("Hello Word"),
            ],
          )),
    ],
  ),
),
```

如上，子元素的大小都会和 `prototypeItem` 中元素的大小进行同步，我们和 SliverFixedExtentList 对比看一下效果

```dart
body: DefaultTextStyle(
  style: TextStyle(fontSize: 60, color: Colors.red),
  child: CustomScrollView(
    slivers: [
      SliverFixedExtentList(
          itemExtent: 40,
          delegate: SliverChildListDelegate(
            [
              Text("Hello Word"),
              Text("Hello Word"),
              Text("Hello Word"),
            ],
          )),
    ],
  ),
),
```

效果如下：

使用 prototype：![image-20210810150249268](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810150249.png),使用 fixed：![image-20210810150332297](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810150332.png)

从图中可以看到，尽管高度固定到 40，但是由于 Text 的大小被修改了，所以渲染出来的还是有问题。

___

#### SliverFillViewport

它也接受一个 delegate，支持动态的加载，只不过内部的子元素会占满整个屏幕

```dart
SliverFillViewport(
    delegate: SliverChildListDelegate([
  Container(color: Colors.red),
  Container(color: Colors.yellow),
  Container(color: Colors.blue),
]))
```

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810151310.gif" alt="345" style="zoom:33%;" />

___

#### SliverAppbar

在 slivers 系列中，SliverAppbar 可以说是使用频率比较高的组件了，SliverAppbar 为应用栏提供了自定义滚动行为，下面我们来看一下

```dart
class _MyHomePageState extends State<MyHomePage> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      drawer: Drawer(),
      body: DefaultTextStyle(
        style: TextStyle(fontSize: 60, color: Colors.red),
        child: CustomScrollView(
          slivers: [
            SliverAppBar(
              title: Text("Sliver AppBar"),
            ),
            SliverToBoxAdapter(child: Placeholder()),
            SliverList(
              delegate: SliverChildListDelegate(
                [
                  FlutterLogo(size: 200),
                  FlutterLogo(size: 200),
                  FlutterLogo(size: 200),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

上面是一个磨人的 SliverAppbar，并没有实现任何特殊效果，默认的效果如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810155126.gif" alt="345" style="zoom:33%;" />

可以看到在滑动的过程中，SliverAppbar 被顶上去了，这也是非常正常的。接着我们来看一下都有哪些特殊效果吧

##### 特殊效果

- floating

  ```dart
  SliverAppBar(
    title: Text("Sliver AppBar"),
    floating: true,
  )
  ```

  在向下滑动的时候，会首先将 SliveAppbar 显示出来，如下：

  <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810155545.gif" alt="345" style="zoom:33%;" />

- pinned ：一直显示在顶部，无视滑动，这样就和普通的导航栏差不多了。区别就是在滑动的时候 SliveAppbar 的底部会有一点点影子

- snap：在滑动停止之后，导航会自动全部显示出来，需要注意的是必须搭配 floating 一起使用，如下：

  ```dart
  SliverAppBar(
    title: Text("Sliver AppBar"),
    snap: true,
    floating: true,
  )
  ```

  <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810160158.gif" alt="345" style="zoom:33%;" />

- flexibleSpace：可展开拉伸的部分

  ```dart
  SliverAppBar(
    // title: Text("Sliver AppBar"),
    expandedHeight: 300,
    stretch: true,
    flexibleSpace: FlexibleSpaceBar(
      background: FlutterLogo(),
      title: Text("FlexibleSpaceBar title"),
      collapseMode: CollapseMode.parallax,
      stretchModes: [
        StretchMode.blurBackground,
        StretchMode.zoomBackground,
        StretchMode.fadeTitle,
      ],
    ),
  ),
  ```

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810165046.gif" alt="345" style="zoom:50%;" />

#### SliverOpacity

透明组件，内部接受的是一个 sliver，所以需要用 SliverToAdapter 转一下

```dart
SliverOpacity(
  opacity: 0.5,
  sliver: SliverToBoxAdapter(
    child: FlutterLogo(
      size: 100,
    ),
  ),
)
```

#### SliverFillRemaining

该组件会填满当前页面的剩余空间

```dart
SliverFillRemaining(
  hasScrollBody: false,
  child: Center(
    child: CircularProgressIndicator(),
  ),
)
```

- hasScrollBody ：当前组件中是否有可滚动的组件

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210810174859.png" alt="image-20210810174859170" style="zoom:33%;" />

### 案例

首先看一下实现的效果(由于是 gif 图，所以看起来有一点卡)：



<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210813165416.gif" alt="345" style="zoom:33%;" />



- 准备数据

  接口来源于网络，仅供学习使用

  ```
  https://h5.48.cn/resource/jsonp/allmembers.php?gid=10
  ```

  对应的数据类：

  ```dart
  class Member {
    final String id;
    final String name;
    final String team;
    final String sid;
    final String gid;
    final String gname;
    final String sname;
    final String fname;
    final String tname;
    final String pid;
    final String pname;
    final String nickname;
    final String company;
    final String join_day;
    final String height;
    final String birth_day;
    final String star_sign_12;
    final String star_sign_48;
    final String speciality;
    final String hobby;
    final String experience;
    final String catch_phrase;
    final String status;
    final String ranking;
    final String tcolor;
    final String gcolor;
  
    String get avatarUrl => "https://www.snh48.com/images/member/zp_$id.jpg";
  
    Member(
      this.id,
      this.name,
     //.....自行添加
    );
  
    @override
    String toString() {
      return "$id  ---  $name";
    }
  }
  ```

- 首页

  ```dart
  class _DemoWidgetState extends State<DemoWidget> {
    List<Member> _member = [];
  
    @override
    Widget build(BuildContext context) {
      return Scaffold(
        appBar: AppBar(
          title: Text("案例"),
        ),
        body: RefreshIndicator(
          onRefresh: () async {
            setState(() => _member.clear());
            final url = "https://h5.48.cn/resource/jsonp/allmembers.php?gid=10";
            final res = await http.get(Uri.parse(url));
            if (res.statusCode != 200) throw Error();
  
            final json = convert.jsonDecode(res.body);
            final members = (json["rows"] as List)
                .map((e) => Member(
                      e['sid'], e["sname"],e["tname"], e["sid"], e["gid"],e["gname"],e["sname"],e["fname"],e["tname"],
                      e["pid"],e["pname"], e["nickname"], e["company"], e["join_day"], e["height"],    e["birth_day"],
                      e["star_sign_12"], e["star_sign_48"], e["speciality"], e["hobby"], e["experience"],
                      e["catch_phrase"],  e["status"], e["ranking"], e["tcolor"],e["gcolor"],
                    ))
                .toList();
  
            setState(() => _member = members);
          },
          child: CustomScrollView(
            slivers: [
              SliverToBoxAdapter(),
              SliverPersistentHeader(
                  delegate: _MyDelegate("SII", Color(0xffae86bb)), pinned: true),
              _buildTeamList("SII"),
              SliverPersistentHeader(
                  delegate: _MyDelegate("NII", Color(0xff91cdeb)), pinned: true),
              _buildTeamList("NII"),
              SliverPersistentHeader(
                  delegate: _MyDelegate("HII", Color(0xffa7b0ba)), pinned: true),
              _buildTeamList("HII"),
              SliverPersistentHeader(
                  delegate: _MyDelegate("预备生", Color(0xff91cdeb)), pinned: true),
              _buildTeamList("预备生"),
              SliverPersistentHeader(
                  delegate: _MyDelegate("荣誉毕业生", Color(0xff8ed2f5)),
                  pinned: true),
              _buildTeamList("荣誉毕业生"),
              SliverPersistentHeader(
                  delegate: _MyDelegate("S预备生", Color(0xff38b26d)), pinned: true),
              _buildTeamList("S预备生"),
              SliverPersistentHeader(
                  delegate: _MyDelegate("X", Color(0xffa7b0ba)), pinned: true),
              _buildTeamList("X"),
            ],
          ),
        ),
      );
    }
  
    SliverGrid _buildTeamList(String teamName) {
      //进行筛选
      final teamMember =
          _member.where((element) => element.team == teamName).toList();
      return SliverGrid(
        delegate: SliverChildBuilderDelegate((context, index) {
          Member m = teamMember[index];
          return InkWell(
            child: Column(
              mainAxisAlignment: MainAxisAlignment.center,
              children: [
                //动画
                Hero(
                    tag: m.avatarUrl,
                    child: ClipOval(
                      child: CircleAvatar(
                        child: Image.network(m.avatarUrl),
                        backgroundColor: Colors.white,
                      ),
                    )),
                Text("${m.name}"),
              ],
            ),
            onTap: () => Navigator.of(context)
                .push(MaterialPageRoute(builder: (_) => DetailPage(m))),
          );
        }, childCount: teamMember.length),
        gridDelegate:
            SliverGridDelegateWithMaxCrossAxisExtent(maxCrossAxisExtent: 120),
      );
    }
  }
  
  
  class _MyDelegate extends SliverPersistentHeaderDelegate {
    final String title;
    final Color color;
  
    _MyDelegate(this.title, this.color);
  
    @override
    Widget build(
        BuildContext context, double shrinkOffset, bool overlapsContent) {
      return Container(
        height: 35,
        child: FittedBox(child: Text(title, style: TextStyle())),
        color: color,
      );
    }
  
    ///最高高度
    @override
    double get maxExtent => 35;
  
    ///最新高度
    @override
    double get minExtent => 35;
  
    ///重绘
    @override
    bool shouldRebuild(covariant _MyDelegate oldDelegate) {
      //如果 title 不相等，则重绘
      return oldDelegate.title != title;
    }
  }
  
  ```

  上面代码在  refresh 中进行了网络请求，然后进行解析数据，最后进行了刷新操作

  上面代码都很简单，不太熟悉的可能就是 `SliverPersistentHeader` 了，这是一个可以置顶的 header，它可以出现在视图的任何一个位置， `pinned` 和 `floating` 属性用来控制收起是是否展示，具体意思和 SliverAppbar 中一样。

- 详情页面

  ```dart
  class DetailPage extends StatelessWidget {
    final Member member;
  
    DetailPage(this.member);
  
    @override
    Widget build(BuildContext context) {
      return Scaffold(
          body: CustomScrollView(
        slivers: [
          SliverAppBar(
              expandedHeight: 300,
              pinned: true,
              stretch: true,
              flexibleSpace: FlexibleSpaceBar(
                centerTitle: true,
                title: Text("${member.name}"),
                background: Center(
                  child: Padding(
                    padding: const EdgeInsets.all(100),
                    //长宽比
                    child: AspectRatio(
                      aspectRatio: 1,
                       // 和上面那个页面的动画对应，tag 必须一致
                      child: Hero(
                        tag: member.avatarUrl,
                        child: Material(
                          elevation: 4.0,
                          shape: CircleBorder(),
                          child: ClipOval(
                            child: Image.network(
                              member.avatarUrl,
                              fit: BoxFit.cover,
                            ),
                          ),
                        ),
                      ),
                    ),
                  ),
                ),
              )),
          SliverList(
              delegate: SliverChildListDelegate(
            [
              _buildInfo("战队：", member.team),
              _buildInfo("公司：", member.company),
              _buildInfo("时间：", member.join_day),
              _buildInfo("身高：", member.height),
              _buildInfo("生日：", member.birth_day),
              _buildInfo("星座：", member.star_sign_12),
              _buildInfo("运势：", member.star_sign_48),
              _buildInfo("爱好：", member.speciality),
              _buildInfo("签名：", member.catch_phrase),
            ],
          ))
        ],
      ));
    }
  
    _buildInfo(String label, String content) {
      return Card(
        child: Padding(
          padding: EdgeInsets.symmetric(vertical: 25),
          child: Row(
            children: [Text(label), Text(content)],
          ),
        ),
      );
    }
  }
  ```

  上面代码中有一个问题，本来使用了  `stretch` 属性之后，在下拉的时候应该会有一个放大的效果，但是运行代码的时候并没有，有知道原因的同学可以讲一下

____

> 参考：B站王叔不秃

> 如果本文有帮助到你的地方，不胜荣幸，如有文章中有错误和疑问，欢迎大家提出!
