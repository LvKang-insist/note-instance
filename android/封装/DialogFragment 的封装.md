​		在日常开发中，我们经常会使用到 对话框。原来一直使用的就是普通的 dialog。但是最近看到了 DialogFragment。于是就研究了一下，他本是也是一个 fragmen，它具有自己的生命周期，只是内部绑定了一个 dialog。经过几天的使用之后，我对他做了一个 封装。满足了日常的各种需求。

​		首先看一下使用方式，共两种使用方式，如下：

​		1，如果你的对话框逻辑非常复杂，里面的控件非常多。你可以使用这种方法。自定义 dialog 继承自 封装好的类。

```java
  DateDialog.MessageBuilder()
                .setContentView(R.layout.dialog)//设置 view
                .setCancelable(false) //设置点击 dialog 以外是否可以取消 dialg
                .setAlpha(1) // 透明度
                .setAutoDismiss(true) //是否禁用所有的点击事件
                .setGravity(Gravity.TOP) //对话框的位置
                .setAnimation(R.style.BottomAnimStyle) // 添加动画
                .build()
                .show(getSupportFragmentManager(),"d1");
```

​		2，如果你的对话框不是太复杂，则可以使用下面这种

```java
BaseFragDialog.Builder()
                .setContentView(R.layout.dialog)
                .setCancelable(false)
                .build()
                .setText(R.id.tv_dialog_title,"我是标题")
                .setText(R.id.tv_dialog_message,"我是内容")
                .setListener(R.id.tv_dialog_cancel, new BaseFragDialog.OnListener() {
                    @Override
                    public void onClick(BaseFragDialog dialog, View view) {
                    }
                })
                .setListener(R.id.tv_dialog_confirm, new BaseFragDialog.OnListener() {
                    @Override
                    public void onClick(BaseFragDialog dialog, View view) {
                    }
                })
                .show(getSupportFragmentManager(),"");
```

