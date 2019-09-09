getChildFragmentManager和getsupportFragmentManager和getFragmentManager的关系

---
- getFragmentManager()所得到的是Fragment的父容器的管理者。
- getChildFragmentManager()所得到的是在fragment里面的子容器的管理者。
- getSupportFragmentManager()主要用于支持3.0以下的版本，3.0以上的可以直接调用getFragmetnMaanager()。因为fragment是3.0以后才出现的组件，为了让之前的的版本也可以使用，所以才有了getSupportFragmentManager();


---
==Fragment嵌套Fragment要用getChildFragmentManager==

容易出错的地方：

- Fragment放ViewPager，ViewPager里面是fragment。第一次进入没问题，再次进入ViewPager的fragment时里面内容就没了,数据丢失
- Fragment低频率点击切换不会发生问题，过快点击马上崩溃
- 错误：java.lang.IllegalArgumentException：No view found for id for fragment
- 调用fragment的replace方法不走onDestroy()、onDestroyView()方法，无法销毁fragment
- 在fragment中写倒计时，每次切换后倒计时越来越快的问题！

---
解决方案

```
Frag7_ViewPager_Adapter adapter = new Frag7_ViewPager_Adapter(
                getChildFragmentManager(),list);
```
getFragmentManager到的是activitry对所包含fragment的Manager，而如果是fragment嵌套fragment，那么就要利用getChildFragmentManager()了。

[原文链接](https://blog.csdn.net/allan_bst/article/details/64920076)