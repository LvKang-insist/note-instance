### 我们经常使用碎片，并且知道他可以动态的替换，但是你知道他可以使用那些方法进行替换，add方法又是怎么使用吗。下面我罗列了一下常用的使用方法。

---
### - relpleac：替换碎片

这个方法在执行的时候会将之前的碎片给清空掉。也就是说当你在用这个方法动态添加碎片的时候，他会将容器里面的所有碎片全部清空，生命周期直接销毁。然后将你要replace的碎片进行显示。
```
getSupportFragmentManager()
                        .beginTransaction()
                        .replace(R.id.content_main, new Fragment_01())
                        .commit();
```
注意：假如说你容器里面只有一个碎片实例，当你在次添加和容器里面实例相同碎片的时候，他不会清除容器里面的碎片，而是使用容器里面的实例。因为两个碎片的实例一样，所以他不会进行添加。

#### 如下所示：

```
case R.id.sliding_btn2:
                if (fragment02 == null) {
                    fragment02 = new Fragment_02();
                    System.out.println("--------------------");
                }
                fragment02.setData(fragment02);
                getSupportFragmentManager().beginTransaction().replace(R.id.content_main, fragment02).commit();
                break;
```
当我点击按钮动态添加碎片，第一次点击按钮他会创建一个碎片对象，进行添加。当第二次点击按钮的时候他使用的是第一次添加时候的实例。如果是这样写的，那么他第二次和以后就不会进行碎片的添加，除非重新new一个碎片的实例，或者清空容器。

---

### - add：添加碎片

这个方法和replace完全不同，他会在容器原有的基础上进行添加。也就是说 当你使用add方法的时候，他不会清空容器内原有的碎片，而是继续添加进去。==有一个问题需要说明，如果你原来显示的碎片是replace进去的，当你再次使用add方法添加的时候，他不会替换原来的碎片，而是会显示在原来碎片的上面。==  
    有两种方法可以解决：
1. 每次动态添加碎片的时候都使用add方法添加，然后就是调用hide和show进行隐藏和显示，hide和show用法会在下面说。
2. 可以想办法获取到原来碎片的实例，然后调用hide方法隐藏原来的碎片，这样就会只显示当前add的碎片。

使用add添加如下所示

```
                    getActivity().
                        getSupportFragmentManager()
                        .beginTransaction()
                        .add(R.id.content_main, frag2_mpchart)
                        .commit();
```
因为我是在activity里面的碎片里面添加碎片所以使用的getActivity。当前这个add这样使用是没有价值的。等会会配合隐藏和显示进行说明。

---

### - hide：隐藏碎片

```
 private void hide(FragmentTransaction transaction){
        if (fragment_02 != null){
            Log.e("demo",""+fragment_02);
            transaction.hide(fragment_02);
        }
        if (frag2_mpchart != null){
            transaction.hide(frag2_mpchart);
        }
    }
```

---
### - show：显示碎片

```
FragmentTransaction transaction = getActivity().
                        getSupportFragmentManager().beginTransaction();
                transaction.add(R.id.content_main,frag2_mpchart).show(frag2_mpchart);
```
这个就代表着 先进行add，然后show。当然，最后肯定还有个提交commit()


---
####  在说怎么使用add的时候，我们先要知道他是用来干什么的。
- 首先，我们有一个需求，现在有一个碎片正在显示，他上面有一个按钮，和一些数据，当我点击按钮的时候切换到别的碎片中，同时我还想让原来碎片上的数据不会丢失。这样可能要用到add动态添加碎片
- 还有一个非常典型的例子。导航栏，当前页面的最下面是好几个按钮，上面则是一个布局，用来显示碎片的内容。当我点击下面导航栏按钮的时候，进行碎片的切换，同时我还想让原来显示的碎片上面的数据不会消失，在这种需求下就可以使用add动态添加。
- 

---
#### add的使用方法：

每次添加碎片的时候都必须让容器里面的碎片全部隐藏，再让你对应的碎片进行显示。对应的就是hide和show方法。
1. 首先，有三个按钮，三个碎片，以及一个要显示碎片的布局。这些我就不多说了。下面分别是布局和点击按钮初始化

```
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity"
    android:orientation="vertical">

    <RelativeLayout
        android:id="@+id/main_content"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="8">
    </RelativeLayout>

    <RelativeLayout
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1">
        <Button
            android:id="@+id/btn_1"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_alignParentLeft="true"
            android:text="1"/>
        <Button
            android:id="@+id/btn_2"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_centerHorizontal="true"
            android:text="2"/>
        <Button
            android:id="@+id/btn_3"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_alignParentRight="true"
            android:text="3"/>
        
    </RelativeLayout>
</LinearLayout>
```

```
 private void init() {
        Button but1 = findViewById(R.id.btn_1);
        Button but2 = findViewById(R.id.btn_2);
        Button but3 = findViewById(R.id.btn_3);
        but1.setOnClickListener(this);
        but2.setOnClickListener(this);
        but3.setOnClickListener(this);
    }
```

2. 然后就是按钮的点击事件：
 
```
@Override
    public void onClick(View v) {
       FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();
        switch (v.getId()) {
            case R.id.btn_1 :
                hideFragment(transaction);
                if (f1 == null) {
                    f1 = new Fragment1();
                    transaction.add(R.id.main_content, f1);
                }
                transaction.show(f1);
                transaction.commit();
                break;
            case R.id.btn_2 :
                hideFragment(transaction);
                if (f2 == null) {
                    f2 = new Fragment2();
                    transaction.add(R.id.main_content, f2);
                }
                transaction.show(f2);
                transaction.commit();

                break;
            case R.id.btn_3 :
                hideFragment(transaction);
                if (f3 == null) {
                    f3 = new Fragment3();
                    transaction.add(R.id.main_content, f3);
                }
                transaction.show(f3);
                transaction.commit();
                break;
            default :
                break;
        }
    }
    
    /**
     * 隐藏所有碎片
     */
    private void hideFragment(FragmentTransaction transaction) {
        if (f1 != null) {
            transaction.hide(f1);
        }
        if (f2 != null) {
            transaction.hide(f2);
        }
        if (f3 != null) {
            transaction.hide(f3);
        }
    }
    
```
 注意每次add 碎片的时候，一定要判 碎片是否为空，为空则new对象，不为空则意味着你已经添加过了，不能再次添加。add多次否则就会报错。
 
 还有每次在使用FragmentTransaction的时候都需要重新获取,每一个FragmentTransaction只能够commit()一次。 
 我这里写的时候就是 每次点击的时候就要重新获取FragmentTransaction。
 
[如果是在碎片的碎片中获取碎片的管理者，请先看一哈这个](https://note.youdao.com/)


---
> 如有错误，还请指出，谢谢！