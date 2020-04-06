该框架时基于碎片的



1，创建一个接口

```java
/**
 * Copyright (C)
 *
 * @file: IWebViewInitializer
 * @author: 345
 * @Time: 2019/5/4 15:32
 * @description: ${DESCRIPTION}
 */
public interface IWebViewInitializer {

    /**
     * 初始化
     */
    WebView initWebView(WebView webView);

    /**
     * 针对浏览器本身行为的一个控制
     */
    WebViewClient initWebViewClient();

    /**
     * 针对页面的 一个控制
     */
    WebChromeClient initWebChromeClient();
}
```

3，创建一个抽象的碎片,

```java
/**
 * Copyright (C)
 *
 * @file: WebDelegate
 * @author: 345
 * @Time: 2019/5/4 15:25
 * @description: ${DESCRIPTION}
 */
public abstract class WebDelegate extends LatteDelegate implements IWebViewInitializer {

    private WebView mWebView = null;
    private final ReferenceQueue<WebView> WEB_VIEW_QUEUE = new ReferenceQueue<>();

    private String mUrl = null;
    /**
     * WebView 的状态
     */
    private boolean mIsWebViewAbailable = false;
    private LatteDelegate mTopDelegate = null;


    public WebDelegate() {
    }

    public abstract IWebViewInitializer setInitializer();


    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        final Bundle args = getArguments();
        mUrl = args.getString(RouteKeys.URL.name());
        initWebView();
    }

    @SuppressLint({"JavascriptInterface", "AddJavascriptInterface"})
    private void initWebView() {
        if (mWebView != null) {
            //调用此方法从ViewGroup中删除所有子视图。
            mWebView.removeAllViews();
            //销毁webView
            mWebView.destroy();
        } else {
            //实例化接口
            final IWebViewInitializer initializer = setInitializer();
            if (initializer != null) {
                //弱引用，避免内存泄露，创建WebView 的对象
                final WeakReference<WebView> webViewWeakReference =
                        new WeakReference<>(new WebView(getContext()), WEB_VIEW_QUEUE);
                mWebView = webViewWeakReference.get();
                //调用接口的方法,
                //初始化WebView
                mWebView = initializer.initWebView(mWebView);
                //处理各种通知，请求事件
                mWebView.setWebViewClient(initializer.initWebViewClient());
                mWebView.setWebChromeClient(initializer.initWebChromeClient());

                final String name = Latte.getConfiguration(ConfigType.JAVASCRIPT_INTERFACE);
                mWebView.addJavascriptInterface(LatteWebInterface.crate(this), name);
                mIsWebViewAbailable = true;
            } else {
                throw new NullPointerException("Initializer is null");
            }
        }
    }

    /**
     *  设置碎片
     */
    public void setTopDelegate(LatteDelegate delegate) {
        mTopDelegate = delegate;
    }
    /**
     * 获取碎片
     */
    public LatteDelegate getTopDelegate(){
        if (mTopDelegate ==null){
            mTopDelegate = this;
        }
        return mTopDelegate;
    }

    /**
     *  获取 webView
     */
    public WebView getWebView() {
        if (mWebView == null) {
            throw new NullPointerException("WebView IS NULL");
        }
        return mIsWebViewAbailable ? mWebView : null;
    }

    public String getUrl() {
        if (mUrl == null) {
            throw new NullPointerException("WebView IS NULL");
        }
        return mUrl;
    }

    @Override
    public void onPause() {
        super.onPause();
        if (mWebView != null) {
            mWebView.onPause();
        }
    }

    @Override
    public void onResume() {
        super.onResume();
        if (mWebView != null) {
            mWebView.onResume();
        }
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        mIsWebViewAbailable = false;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        if (mWebView != null) {
            mWebView.removeAllViews();
            mWebView.destroy();
            mWebView = null;
        }
    }
}
```

其中 LatteDelegate 是一个基类碎片。

3,创建一个类，来实现这个抽象类

```java
/**
 * Copyright (C)
 *
 * @file: WebDelegateImpl
 * @author: 345
 * @Time: 2019/5/4 16:01
 * @description: ${DESCRIPTION}
 */
public class WebDelegateImpl  extends WebDelegate{

    private IPageLoadListener mIPageLoadListener = null;
    /**
     * @param url webView 的显示的内容地址
     * @return 放回当前碎片的对象。
     */
    public static WebDelegateImpl create(String url){
        final Bundle args= new Bundle();
        args.putString(RouteKeys.URL.name(),url);
        final WebDelegateImpl delegate = new WebDelegateImpl();
        delegate.setArguments(args);
        return delegate;
    }
    @Override
    public Object setLayout() {
        return getWebView();
    }

    public void setPageLoadListener(IPageLoadListener loadListener){
        this.mIPageLoadListener = loadListener;
    }

    @Override
    public void onBindView(@Nullable Bundle savedInstanceState, View rootView) {
        if (getUrl() != null){
            //用原生的 方式模拟web跳转并进行页面加载
            Router.getInstance().loadPage(this,getUrl());
        }
    }

    /**
     * @return 返回IWebViewInitializer 接口的实例
     */
    @Override
    public IWebViewInitializer setInitializer() {
        return this;
    }

    // 实现 IWebViewInitializer 接口的三个方法,
    /**
     * 初始化WebView
     */
    @RequiresApi(api = Build.VERSION_CODES.KITKAT)
    @Override
    public WebView initWebView(WebView webView) {
        return new WebViewInitializer().createWebView(webView);
    }
    /**
     *  处理webView 的各种事件
     */
    @Override
    public WebViewClient initWebViewClient() {
        final WebViewClientImpl client = new WebViewClientImpl(this);
        //传入 IPageLoadListener 接口的实例
        client.setPageLoadListener(mIPageLoadListener);
        return client;
    }

    @Override
    public WebChromeClient initWebChromeClient() {
        return new WebChromeClientImpl();
    }
}
```