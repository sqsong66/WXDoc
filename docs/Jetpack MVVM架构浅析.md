## 概述
整体框架采用`Kotlin+Hilt+Retrofit+OkHttp+Coroutines`。采用`Hilt`来进行对象的依赖注入，`OkHttp`、`Retrofit`进行网络请求，`Coroutines`进行异步操作(网络请求、数据库处理、密集计算等)。

## 基础信息封装
### 1、`BaseApplication`封装
`BaseApplication`需要使用`@HiltAndroidApp`注解，方便后续`Hilt`对相应的对象进行依赖管理。
```kotlin
@HiltAndroidApp
class BaseApplication: Application() {
    override fun onCreate() {
        super.onCreate()
        instance = this
    }

    companion object {
        private lateinit var instance: BaseApplication
        fun getInstance(): BaseApplication = instance
    }
}
```

### 2、全局实例使封装
网络请求实例`ApiService`对象创建。使用`Hilt`来提供对象的类需要使用`@Module`注解，同时，使用`@InstallIn`注解来标识该类所具有的生命周期。网络请求属于全局类型，可以使用`Hilt`提供的`SingletonComponent`来表示其全局单例属性。\
所有提供对象的方法需要使用`@Provides`注解， \
`@Singleton`注解标识该方法提供的实例为单例对象。
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object CommonModule {
    ...
    @Singleton
    @Provides
    fun provideApiService(): ApiService {
        return Retrofit.Builder()
            .client(OkHttpClientFactory.createUnsafeOkHttpClient())
            .addConverterFactory(GsonConverterFactory.create())
            .baseUrl(ApiService.BASE_URL)
            .build()
            .create(ApiService::class.java)
    }
    ...
}
```
`ApiService`中的接口函数使用`suspend`关键字修饰标识为挂起函数，方便后续在协程中调用。
```kotlin
interface ApiService {

    @GET("article/list/{page}/json")
    suspend fun getHomeArticles(@Path("page") page: Int): BaseBean<ArticleData>

    @GET("banner/json")
    suspend fun getHomeBanner(): BaseBean<List<BannerData>>

    companion object {
        const val BASE_URL = "https://www.wanandroid.com/"
    }

}
```

### 3、基类封装
`BaseActivity`:
```kotlin
abstract class BaseActivity<V : ViewDataBinding> : AppCompatActivity() {

    protected lateinit var binding: V

    // init data binding.
    abstract fun initBinding(): V

    // init your something...
    abstract fun initEvent(savedInstanceState: Bundle?)

    // define the status bar text color(black or white).
    open fun isDarkStatusBarText(): Boolean = true

    open fun preInflateView() {}

    override fun onCreate(savedInstanceState: Bundle?) {
        // Set status bar text color.
        setStatusBarTextColor(isDarkStatusBarText())
        preInflateView()
        super.onCreate(savedInstanceState)
        binding = initBinding()
        setContentView(binding.root)
        initEvent(savedInstanceState)
    }

    override fun onOptionsItemSelected(item: MenuItem): Boolean {
        if (item.itemId == android.R.id.home) finish()
        return super.onOptionsItemSelected(item)
    }

}
```
`BaseFragment`:
```kotlin
abstract class BaseFragment<V: ViewDataBinding>: Fragment() {

    protected lateinit var binding: V

    // init data binding.
    abstract fun initBinding(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): V

    // init your something...
    abstract fun initEvent()

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        binding = initBinding(inflater, container, savedInstanceState)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        initEvent()
    }

}
```

## 界面示例
使用`Hilt`注解后，Android中的四大组件需要使用`@AndroidEntryPoint`注解，这样在四大组件中使用其他对象时也可以使用注解来进行对象的依赖注入。
```kotlin
@AndroidEntryPoint
class MainActivity : BaseActivity<ActivityMainBinding>() {

    private val mainViewModel by viewModels<MainViewModel>()

    override fun initBinding(): ActivityMainBinding = ActivityMainBinding.inflate(layoutInflater)

    override fun initEvent(savedInstanceState: Bundle?) {
        val navController = findNavController(R.id.nav_host_fragment_activity_main)
        binding.navView.setupWithNavController(navController)

        // 请求banner数据
        mainViewModel.requestBannerData()

        mainViewModel.bannerList.observe(this) {
            showToast("Banner size: ${it.size}")
        }
    }

}
```
通过Kotlin扩展函数特性，在`Activity`中获取`ViewModel`对象便变得更加简单了。并且`ViewModel`的生命周期也与`Activity`进行了绑定。

```kotlin
@HiltViewModel
class MainViewModel @Inject constructor() : ViewModel() {

    @SuppressLint("StaticFieldLeak")
    @Inject
    @ApplicationContext
    lateinit var context: Context

    @Inject
    lateinit var apiService: ApiService

    val bannerList = MutableLiveData<List<BannerData>>()

    fun requestBannerData() {
        // ViewModel扩展函数
        requestDataFromServer({ apiService.getHomeBanner() }, {
            if (it.isSuccess) {
                val resultList = it.getOrNull()
                if (resultList != null) {
                    bannerList.postValue(resultList)
                }
            } else {
                context.showToast(it.exceptionOrNull()?.showMessage)
            }
        })
    }

}
```
`Hilt`中，也有一个专门的`@HiltViewModel`注解专门用来注释`ViewModel`，这样在`ViewModel`中便能使用注解的方式来获取到相应的依赖对象了。\
`requestDataFromServer`函数是对`ViewModel`进行扩展的一个函数，为了方便进行网络请求的处理。
```kotlin
/** 普通从服务器请求数据结果。
 * [block] 请求接口挂起函数；该函数会在IO子线程中执行，返回的结果是[BaseBean]子类型
 * [result] 请求结果回调函数；
 *  */
fun <T> ViewModel.requestDataFromServer(
    block: suspend () -> BaseBean<T>,
    result: (ResponseResult<T?>) -> Unit
) {
    viewModelScope.launch {
        kotlin.runCatching {
            withContext(Dispatchers.IO) {
                val blockResult = block()
                if (blockResult.errorCode == 0) {
                    blockResult.data
                } else {
                    throw ApiException(blockResult.errorCode, blockResult.errorMsg, blockResult.errorMsg)
                }
            }
        }.onSuccess {
            result(ResponseResult.success(it))
        }.onFailure {
            it.printStackTrace()
            result(ResponseResult.failure(it))
        }
    }
}
```
`viewModelScope`是`ViewModel`内置的一个关于协程作用域的成员变量。使用它开启一个协程，进行网络请求的耗时操作。这样我们并不需要关注`Activity`的生命周期，因为`ViewModel`与`Activity`的生命周期是相绑定的，在`Activity`销毁的时候，`ViewModel`会做相应的清理工作。这样一个请求就变得非常简单了。

完整的demo示例地址：https://github.com/sqsong66/JetpackMVVMSample

示例中还展示了Paging3来进行分页列表数据请求，相比传统的分页列表数据处理显得更加简洁简单。这里不再赘述，感兴趣的小伙伴可以下载demo示例查看。