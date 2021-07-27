# **全局配置**
抠图的全局配置主要是在`GlobalApplication`中。 `GlobalApplication`中的`initModel()`方法是用来初始化中台(公共库)这各库的。


## **初始化各模块**
目前许多的公共模块是在`CommonBusinessApplication`中进行初始化的，外部通过`Builder`模式构建好相关模块参数设置给`CommonBusinessApplication`在其内部进行初始化。


### **日志(Log)模块**
主要是通过构建`LogBuilder`，设置日志标签。
```java
// 日志参数
LogBuilder logBuilder = new LogBuilder();
logBuilder.setTag("AiCut");
```


### **ServerFacade模块**
该模块位于`commonLib`库的`ServerFacade`中，数据的配置在`ServerFacadeApplication`类中。 该模块的功能主要有：

- 意见反馈日志上报；
- `OkHttp`初始化(设置`UserAgent`、请求头设置IP地址)；
- APP升级检测。

在`GlobalApplication`中主要是通过构建`ServerFacedBuilder`来设置以上功能所需要的各PO线产品对应的参数信息。  

- **`setProId()`**: 设置产品id，抠图对应的是367；
- **`setAppName()`**: 设置APP名称-**傲软抠图**；
- **`setLogoResource()`**: 设置产品Logo资源。

```java
ServerFacedBuilder serverFacedBuilder = new ServerFacedBuilder();
// 设置产品id,这里用来统计新装/活跃、升级、feedback
serverFacedBuilder.setProId(AppNameMap.APP_ID_BackgroundEraser)
        .setAppName(getString(R.string.key_lunch_appname))
        .setLogoResource(R.mipmap.ic_logo_small);
```


### **归因模块**
归因模块主要是设置APP的名称，以及归因SDK的秘钥。
```java
FlyerBuilder flyerBuilder = new FlyerBuilder();
// 设置产品id,这里用来归因和上报转化事件
flyerBuilder.setProId(AppNameMap.APP_ID_BackgroundEraser) 
    // AppsFlyer秘钥
    .setAfDevKey("xxxxxxxxxx");
```


### **`OkHttp`参数初始化**
`OkHttp`的实际是在归因模块初始化的，但其内部初始化比较简单(主要是设置`UserAgent`和在请求头中添加ip参数，参见`ServerFacadeApplication`中的`init()`方法)。  
在抠图业务中，添加了一些额外的信息：

- **日志拦截器**：`HttpLoggingInterceptor`，方便调试请求信息；
- **业务请求头**：`RequestHeaderInterceptor`。  
    1. 请求头中添加授权`token`，后台鉴权用；
    2. 请求头中添加`wx-real-ip`、`User-Agent`，后台排查问题用；
    3. 拦截401`token`失效状态码，需要退出登录，重新获取鉴权`token`。
- **切换域名**：通过`RetrofitUrlManager`进行域名切换。该方式主要是对新封装的`Retrofit`+`OkHttp`请求生效。通过`RetrofitUrlManager`将`baseUrl`切换成项目业务线的。

```java
HttpLoggingInterceptor logging = new HttpLoggingInterceptor();
logging.setLevel(BuildConfig.DEBUG ? HttpLoggingInterceptor.Level.BODY : HttpLoggingInterceptor.Level.NONE);
OkHttpUtils.addInterceptor(logging);
OkHttpUtils.addInterceptor(new RequestHeaderInterceptor());
RetrofitUrlManager.getInstance().setDebug(BuildConfig.DEBUG);
RetrofitUrlManager.getInstance().putDomain(ApiConstant.KEY_BASE_URL_USER, ApiConstant.BASE_URL_USER);
```

???+ Attention "注意"  
    目前项目中使用了两套网络请求框架：  
    1. `OkHttpUtils`：对原生`OkHttp`的封装；目前项目中大部分请求使用的是这种方式。  
    2. `RetrofitClient`：新封装的基于`OkHttp`+`Retrofit`，目前项目中是在该基础上配合`RxJava`来进行网络请求的。该封装后期也可以配合`Kotlin`的协程(Coroutines)进行网络请求。 


### **`CommonBusinessApplication`初始化**
`CommonBusinessApplication`里面主要配置各个模块的业务参数，然后在其内部统一初始化个业务模块(归因、日志、埋点、网络请求等)。
```java
// 敏感业务，是否需要同意隐私政策弹框后初始化
boolean isInitAfterAgreePrivacy = InitConfig.TermsDialogHelper.shouldShowTermsDialog(getApplicationContext());
CommonBusinessApplication commonBusinessApplication = CommonBusinessApplication.getInstance();
commonBusinessApplication.applicationOnCreate(this)
    // 设置日志
    .setLogBuilder(logBuilder)
    // 设置ServerFaced参数
    .setServerFacedBuilder(serverFacedBuilder)
    // 设置归因参数
    .setFlyerBuilder(flyerBuilder, mFlyerCallback)
    //是否需要同意隐私政策弹框后，初始化敏感业务
    .setInitAfterAgreePrivacy(isInitAfterAgreePrivacy)
    .init();
```
???+ Note  
    此处有一个配置参数`isInitAfterAgreePrivacy`,该参数主要是用来控制，当第一次安装应用时，会弹出隐私协议对话框，只有当用户统一了隐私协议之后才会去初始化一些敏感业务。如获取Android_ID、手机设备信息等(目前应用市场对获取用户隐私权限这一块审核比较严)。

???+ Attention "注意"  
    `OkHttp`参数初始化需要放到`CommonBusinessApplication`初始化之前，因为`OkHttp`对象的实例化是在`CommonBusinessApplication`中的`initServerFacade()`方法初始化的，因此在实例化`OkHttp`必须要设置好其参数(`interceptor`)，否则将会出现设置失效。


### **`OkHttp`参数初始化**