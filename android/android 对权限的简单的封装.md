android 动态权限的简单封装。

在android 6.0 以后，如果你要进行一些敏感的操作，你就必须动态的申请权限。

在动态申请权限的时候必须先加入静态的权限，在清单文件中进行注册。

```java
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

然后就可以进行动态权限的请求了。

------

因为权限必须在Activity 中进行申请，为了 单一原则，我将他封装在一个抽象的Activity中，然后让别的activity都继承他。

```java
@SuppressWarnings("ALL")
public abstract class PermissionCheckPepsi extends BaseActivity {

    private  ICheckPermission mICheckPermission = null;

    interface ICheckPermission {
        void onAllow();

        void onReject();
    }
    
    public void checkPermission(String[] permission, ICheckPermission iCheckPermission) {
        if (Build.VERSION.SDK_INT < 23 || permission.length == 0) {
            if (iCheckPermission != null){
                iCheckPermission.onAllow();
            }
        }else {
            if (iCheckPermission != null){
                mICheckPermission = iCheckPermission;
            }
            ActivityCompat.requestPermissions(this,permission,0);
        }
    }

    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        if (mICheckPermission != null && requestCode == 0){
            for (int i = 0; i < grantResults.length; i++) {
                //判断权限是否被允许，只要又一次拒绝就算失败
                if (grantResults[i] == PackageManager.PERMISSION_DENIED){
                    // 1：用户拒绝了该权限，没有勾选"不再提醒"，此方法将返回true。
                    // 2：用户拒绝了该权限，有勾选"不再提醒"，此方法将返回 false。
                    // 3：如果用户同意了权限，此方法返回false
                    if (!ActivityCompat.shouldShowRequestPermissionRationale(this,permissions[i])){
                        Toast.makeText(this, "你已拒绝此权限，如果需要，可以在设置中打开此权限", Toast.LENGTH_SHORT).show();
                    }else {
                        mICheckPermission.onReject();
                    }
                    return;
                }
            }
            mICheckPermission.onAllow();
        }
    }
}
```

如上就是一个简单的封装,如果要使用，只需要继承这个类直接调用就好了。

```java
image.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        checkPermission(new String[]{Manifest.permission.WRITE_EXTERNAL_STORAGE}, new ICheckPermission() {
            @Override
            public void onAllow() {
                Toast.makeText(MainActivity.this, "成功", Toast.LENGTH_SHORT).show();
            }
            @Override
            public void onReject() {
                Toast.makeText(MainActivity.this, "失败", Toast.LENGTH_SHORT).show();
            }
        });
    }
});
```



注意：要遵循单一原则。要保证一个类只干一件事，像申请权限，直接弄一个抽象类进行封装，让别的类继承自他，这种方式体验非常好。