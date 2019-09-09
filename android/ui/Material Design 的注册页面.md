Material Design 是由谷歌的实际工程师鞥基于传统优秀的设计原则，发明的一套全新的界面的设计语言，非常好看

1，首先添加依赖

```java
api 'com.android.support:design:28.0.0'
```

2,布局文件

```java
<?xml version="1.0" encoding="utf-8"?>
<android.support.v7.widget.LinearLayoutCompat xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <android.support.v7.widget.Toolbar
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="@android:color/holo_orange_dark">

        <android.support.v7.widget.AppCompatTextView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:gravity="center"
            android:text="@string/login"
            android:textColor="@android:color/white"
            android:textSize="20sp" />
    </android.support.v7.widget.Toolbar>

    <android.support.v4.widget.NestedScrollView
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <android.support.v7.widget.LinearLayoutCompat
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:fitsSystemWindows="true"
            android:orientation="vertical"
            android:paddingLeft="24dp"
            android:paddingTop="56dp"
            android:paddingRight="24dp">

            <android.support.v7.widget.AppCompatImageView
                android:layout_width="wrap_content"
                android:layout_height="72dp"
                android:layout_gravity="center_horizontal"
                android:layout_marginBottom="24dp"
                android:src="@mipmap/ic_launcher" />

            <!-- 姓名-->
            <android.support.design.widget.TextInputLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:layout_marginBottom="8dp">

                <android.support.design.widget.TextInputEditText
                    android:id="@+id/edit_sign_up_name"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:hint="@string/login_name"
                    android:inputType="textPersonName" />
            </android.support.design.widget.TextInputLayout>

            <!-- 邮箱-->
            <android.support.design.widget.TextInputLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:layout_marginBottom="8dp">

                <android.support.design.widget.TextInputEditText
                    android:id="@+id/edit_sign_up_email"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:hint="@string/login_email"
                    android:inputType="textEmailAddress" />
            </android.support.design.widget.TextInputLayout>


            <!-- 手机号码-->
            <android.support.design.widget.TextInputLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:layout_marginBottom="8dp">

                <android.support.design.widget.TextInputEditText
                    android:id="@+id/edit_sign_up_phone"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:hint="@string/login_phone"
                    android:inputType="phone" />
            </android.support.design.widget.TextInputLayout>


            <!-- 密码-->
            <android.support.design.widget.TextInputLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:layout_marginBottom="8dp">

                <android.support.design.widget.TextInputEditText
                    android:id="@+id/edit_sign_up_passwrod"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:hint="@string/login_password"
                    android:inputType="textPassword" />
            </android.support.design.widget.TextInputLayout>

            <!-- 重复密码-->
            <android.support.design.widget.TextInputLayout
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:layout_marginBottom="8dp">

                <android.support.design.widget.TextInputEditText
                    android:id="@+id/edit_sign_up_re_password"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:hint="@string/login_rep_password"
                    android:inputType="textPassword" />
            </android.support.design.widget.TextInputLayout>

            <android.support.v7.widget.AppCompatButton
                android:id="@+id/btn_sign_up"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_marginTop="24dp"
                android:layout_marginBottom="24dp"
                android:background="@android:color/holo_orange_dark"
                android:gravity="center"
                android:padding="12dp"
                android:text="@string/login"
                android:textColor="@android:color/white" />

            <android.support.v7.widget.AppCompatTextView
                android:id="@+id/tv_link_sign_in"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:layout_gravity="center"
                android:layout_marginBottom="24dp"
                android:gravity="center"
                android:text="@string/please_login"
                android:textSize="16sp" />
        </android.support.v7.widget.LinearLayoutCompat>



    </android.support.v4.widget.NestedScrollView>

</android.support.v7.widget.LinearLayoutCompat>
```