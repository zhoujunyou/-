### 美味Android Hybrid学习

第一次接共享餐厅管理系统 这里记录一下

Hybird框架代码在cn.mwee.hybrid:core lib下



######Android 和JS是如何通信的 

WebView调用JS有以下两种方式:

- 通过WebView.loadUrl()
- 通过WebView.evaluateJavascript()

在API 19之前是只能通过WebView.loadUrl()进行调用JavaScript。在API 19的时候新提供了WebView.evaluateJavascript()，它的运行效率会比loadUrl()高，还可以传入一个回调对象，方便获取Web端的回传信息

###### JS调用Android代码

JS调用Native代码有以下三种方式：

- 通过WebView.addJavascriptInterface()
- 通过WebViewClient.shouldOverrideUrlLoading()
- 通过WebChromeClient.onJsAlert()、onJsConfirm()、onJsPrompt()

WebView.addJavascriptInterface()是官方推荐的做法，在默认情况下WebView是关闭了JavaScript的调用，需要调用WebSetting.setJavaScriptEnabled(true)来进行开启。这个方法需要一个Object类型的JavaScript Interface，然后通过@JavascriptInterface来标注提供的JS调用的方法

1. 提供用于JS调用的方法必须为public类型
2. 在API 17及以上，提供用于JS调用的方法必须要添加注解@JavascriptInterface
3. 这个方法不是在主线程中被调用的



---

需要通信的方法先定义在配置文件中。

```json
{
  "version": "0.0.1",
  "versionDescription": "beta1",
  "author": "ai.quantong@puscene.com",
  "description": "this is the protocol between JS and native in mwee",
  "appScheme": "meiwei",
  "host": "wireless",
   //用于native 执行js 调用的js方法 
  "jsCallBack": "MWJSBridge.nativeCallback",
    //js调用native 路由配置 第一层为各个模块 nv_module为类名 func_route下对应方法名
  "route": {  
    "authorization": {
      "nv_module": "cn.mwee.hybrid.api.controller.auth.AuthController",
      "func_route": {
        "get_user_token": "getUserToken",
        "call_native_login": "callNativeLogin",
        "call_weixin_login": "callWeixinLogin"
      }
    },
    "navigator_bar": {
      "nv_module": "cn.mwee.hybrid.api.controller.navigator.NavBarController",
      "func_route": {
        "set_nav_bar_left": "setNavBarLeft",
        "set_nav_bar_center": "setNavBarCenter",
        "set_nav_bar_right": "setNavBarRight",
        "hide_nav_bar": "hideNavBar",
        "show_nav_bar": "showNavBar"
      }
    },
    "ui": {
      "nv_module": "cn.mwee.hybrid.api.controller.ui.UiController",
      "func_route": {
        "show_loading": "showLoading",
        "hide_loading": "hideLoading",
        "jump_to_tabbar": "jumpTabbar"
      }
    },
    "pay": {
      "nv_module": "cn.mwee.hybrid.api.controller.pay.PayController",
      "func_route": {
        "call_wx_pay": "callWxPay",
        "call_ali_pay": "callAliPay",
        "call_native_pay": "nativePay"
      }
    },
    "location": {
      "nv_module": "cn.mwee.hybrid.api.controller.location.LocationController",
      "func_route": {
        "get_location": "getLoaction",
        "get_current_city": "getCurrentCity"
      }
    },
```

Application中初始化 主要功能是收集配置文件中的模块及方法信息保存到ActionMapping

构建一个PageManager关联activity生命周期保存页面信息。添加通信拦截器，设置debug模式。

```java
 HybridConfig hybridConfig = new HybridConfig.Builder()
                .setExtConfigId(R.raw.mw_hybrid_config_ext)//扩展路由的配置文件
                .setDebug(!BaseConfig.isProduct())//开启debug模式
                .build();
        Hybrid.init(this, hybridConfig);
```



加载WebView时外面包一层WebFragment. 可以传递以下参数到webfragment

```java
public static Bundle createBundle(HybridParams hybirdParams) {
    Bundle bundle = new Bundle();
    if (hybirdParams != null) {
        bundle.putString(Constant.ARG_URL, hybirdParams.getUrl());
        bundle.putBoolean(Constant.ARG_SHOW_LOADING, hybirdParams.isShowLoading());
        bundle.putBoolean(Constant.ARG_TOP_BAR_VISIBLE, hybirdParams.isTopbarVisible());
        bundle.putString(Constant.ARG_TITLE, hybirdParams.getTitle());
        bundle.putBoolean(Constant.ARG_SHOW_BACK, hybirdParams.isShowBack());
        bundle.putSerializable(Constant.ARG_SHARE_PARAMS, hybirdParams.getShareBean());
        bundle.putString(Constant.ARG_TAG, hybirdParams.getTag());
    }
    return bundle;
}

 webFragment = HybridStarter.createBuilder()
                .setHybridSetttingClassName(MyHybridSetting.class.getName())
                .createFragmentStarter(getIntent().getExtras())
                .setManager(getSupportFragmentManager())
                .setContainerId(R.id.web_fragment_container)
                .setFragmentClass(HybridWebFragment.class)
                .startFragment();
```

WebFragment在onCreateView 中加载默认布局.需要修改一些布局可以复写HybridSetting类里的方法实现

```java
@Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        webLayout = (WebLayout) inflater.inflate(R.layout.hybrid_fragment, null);
        stateView = (StatusView) inflater.inflate(R.layout.hybrid_content_layout, null);
        topbarLayout = webLayout.findViewById(R.id.topbarLayout);
        contentLayout = webLayout.findViewById(R.id.contentLayout);
        refreshLayout = getRefreshLayoutFactory().createRefreshLayout(getActivity(), stateView);
        contentLayout.addView(refreshLayout.getRefreshLayout(), new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
        webView = stateView.findViewById(R.id.webView);
        progressBar = webLayout.findViewById(R.id.progressBar);
        progressBar.setVisibility(View.GONE);


        initTopBar();
        initErrorView();
        initRefreshLayout();
        //webView的一些配置
        initWebView();
        initConnectDetector();
        return webLayout;
    }
```

onViewCreated 在Hybride中注册该IContainer  WebFragment实现了IContainer接口.modules里就是配置文件中的内容，

JNInterface中的call方法就是JS到Native调用的方法

```java
public static void register(IContainer container){
    if(container.getWebView() == null){
        throw new RuntimeException("请确保container.getWebView()不能为null");
    }
    List<String> modules = new ArrayList<String>();
    modules.addAll(me.sdkActionMapping.getModules());
    modules.addAll(me.extActionMapping.getModules());
    for(String module : modules){
        if(!TextUtils.isEmpty(module)){
            JNInterface jnInterface = new JNInterface(container);
            container.getWebView().addJavascriptInterface(jnInterface, getInterfaceName(module));
        }
    }
}

final class JNInterface {

    private IContainer container;

    JNInterface(IContainer container) {
        this.container = container;
    }

    @JavascriptInterface
    @UiThread
    public void call(String path, String params) {
        JSBridge.parseToJNRequest(container, path, params);
    }


```

 onActivityCreated 中调用load方法.用拦截器的方式去处理类似okhttp中的处理方式。最终调用webview的loadurl方法 

```java
private void load() {
    if (showLoading || isReLoadWithLoading) {
        showLoading();
    }
    WebViewLoadUrlInterceptor interceptor = new WebViewLoadUrlInterceptor(webViewHelper);
    List<LoadUrlInterceptor> interceptors = new ArrayList<LoadUrlInterceptor>();
    interceptors.addAll(Hybrid.getLoadUrlInterceptors());
    interceptors.add(interceptor);
    RealLoadUrlInterceptorChain chain = new RealLoadUrlInterceptorChain(getActivity(), tag, url, interceptors, 0);
    try {
        chain.proceed(url);
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

onResume 中的pushResume看看native如何调用js.excuteJs方法中调用 webView.evaluateJavascript方法或者webView.loadUrl。 Hybrid.getJsCallBack()获取的就是在配置文件中的MWJSBridge.nativeCallback

```java
Hybrid.with(getWebView()).njPush(JSBasePush.METHOD_RESUME).push();

 public static JSBridge with(WebView webView){
        return new JSBridge(webView);
    }
  public NJBridge njPush(String method){
        return new NJBridge(webView, method);
    }
public void push(){
        if(EmptyUtils.isEmpty(request.getData())){
            request.setData(new Object());
        }
        JSBridge.njRequest(webView, request);
    }
 @Override
    public void nj(NJChain chain) throws Exception {
        final WebView webView = chain.webView();
        final NJRequest request = chain.request();
        JSBridge.processNJRequest(webView, request, new ValueCallback<String>() {
            @Override
            public void onReceiveValue(String value) {
                Logger.d("nj-onReceiveValue: "+value);
                njCallback(value, webView, request);
            }
        });
    }

 private static void executeJs(WebView webView, String js, final ValueCallback<String> callback) {
        if(webView == null){
            return;
        }
        String executableJs = Hybrid.getJsCallBack() + "(\"" + js + "\")";
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            webView.evaluateJavascript("javascript:" + executableJs, callback);
        } else {
            webView.loadUrl("javascript:" + executableJs);
        }
    }
```

接受JS回调处理  JSBridge.parseToJNRequest(container, path, params); 调用到Hybrid的callNative方法

通过module和method从ActionMap中找到Action。实例化action.getModuleKey()类controller 。从controller中找到action.getMethodKey()方法method 。判断方法上的ActionKey注解值是否与method方法名相同.如果存在执行该方法。否则给js反馈不存在该方法。

``` java
static void parseToJNRequest(final IContainer container, final String path, final String params){
    container.getActivity().runOnUiThread(new Runnable() {
        @Override
        public void run() {
            JNRequest request = new JNRequest();
            try {
                JSONObject jsonObject = new JSONObject(params);
                request.setSerial(jsonObject.optString("cbId", null));
                request.setModule(path);
                request.setMethod(jsonObject.optString("method", null));
                request.setData(jsonObject.optString("apiParams", null));
            } catch (JSONException e) {
                e.printStackTrace();
            }

            try {
                List<HybridInterceptor> list = interceptors();
                RealInterceptorBeforeChain chain = new RealInterceptorBeforeChain(container, request, list, 0);
                chain.proceed(request);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    });
}

 @Override
    public void jnBefore(BeforeChain chain) throws Exception {
        IContainer container = chain.container();
        JNRequest request = chain.request();
        Hybrid.jsCallNative(container, request);
    }
```



```java
private void callNative(IContainer container, JNRequest request){
    Invocation invocation = null;
    if(extActionMapping != null){//从扩展表中找Action
        Action action = extActionMapping.getAction(request.getModule(), request.getMethod());
        invocation = getInvocation(container, action, request);
    }
    if(invocation == null){//从SDK表中找Action
        Action action = sdkActionMapping.getAction(request.getModule(), request.getMethod());
        invocation = getInvocation(container, action, request);
    }
    if(invocation != null){
        try {
            invocation.invoke();
        }catch (Exception e){
            e.printStackTrace();
            Hybrid.with(container.getWebView())
                    .njResponse(request)
                    .errorCode(Hybrid.RESULT_CODE_ERROR)
                    .errorMsg("方法调用发生异常")
                    .res();
        }
        return;
    }

    Hybrid.with(container.getWebView())
            .njResponse(request)
            .errorCode(Hybrid.RESULT_CODE_INVALID_METHOD)
            .errorMsg(String.format("native中不存在方法:%s/%s", request.getModule(), request.getMethod()))
            .res();
}
```



上面是简单的理解大致流程。 其中还有挺多的细节。主要就是两边定协议。