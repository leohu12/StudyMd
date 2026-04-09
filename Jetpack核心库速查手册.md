# Android Jetpack 核心库速查手册

> 面向有经验的 Android 开发者 | 基于 developer.android.com 官方文档整理

---

## 目录

- [一、架构类 (Architecture)](#一架构类-architecture)
  - [1. ViewModel](#1-viewmodel)
  - [2. LiveData](#2-livedata)
  - [3. Room](#3-room)
  - [4. DataStore](#4-datastore)
  - [5. Navigation](#5-navigation)
  - [6. Paging 3](#6-paging-3)
  - [7. Hilt](#7-hilt)
- [二、UI 类](#二ui-类)
  - [8. Jetpack Compose](#8-jetpack-compose)
  - [9. Material 3](#9-material-3)
  - [10. Compose Navigation](#10-compose-navigation)
  - [11. ViewBinding](#11-viewbinding)
  - [12. Fragment](#12-fragment)
  - [13. RecyclerView](#13-recyclerview)
- [三、基础类 (Foundation)](#三基础类-foundation)
  - [14. WorkManager](#14-workmanager)
  - [15. App Startup](#15-app-startup)
  - [16. SplashScreen](#16-splashscreen)
- [四、行为类 (Behavior)](#四行为类-behavior)
  - [17. Biometric](#17-biometric)
  - [18. CameraX](#18-camerax)
  - [19. Media3](#19-media3)
  - [20. Notifications](#20-notifications)
- [速查总表](#速查总表)

---

## 一、架构类 (Architecture)

### 1. ViewModel

**一句话定位：** 屏幕级的状态容器，旋转屏幕数据不丢。

**解决了什么问题：**
Activity 旋转一下就被销毁重建，之前的数据全没了。以前你得自己 `onSaveInstanceState` 手动存取，又麻烦又容易漏。ViewModel 就是来干这个的——它活得比 Activity 长，旋转时数据自动保留。

**核心原理：**

打个比方：Activity 是工位，ViewModel 是你工位上锁着的抽屉。工位被拆了重建（旋转），但抽屉还在，新工位打开同一个抽屉，东西一样不少。

技术层面：
- ViewModel 的生命周期绑定到 `ViewModelStoreOwner`（Activity、Fragment、NavBackStackEntry），不是绑定到 Activity 本身
- 内部通过 `ViewModelStore`（一个 HashMap）存所有 ViewModel 实例，Key 是类名
- 旋转时 Activity 重建，但 `ViewModelStore` 通过 `NonConfigurationInstances` 保留下来
- Activity 真正 finish 时才调 `onCleared()` 清理
- `SavedStateHandle` 解决进程被杀后的数据恢复问题（比 ViewModelStore 更持久）

**关键 API：**

| API | 用途 |
|-----|------|
| `ViewModel` | 基类，放状态和业务逻辑 |
| `ViewModelProvider` | 创建/获取 ViewModel |
| `viewModels()` | Kotlin 属性委托，一行拿到 ViewModel |
| `SavedStateHandle` | 进程重建后恢复数据 |
| `viewModelScope` | 内置协程作用域，跟 ViewModel 同生共死 |

**代码示例：**

```kotlin
// 定义 ViewModel
class UserViewModel(savedStateHandle: SavedStateHandle) : ViewModel() {
    // 用 StateFlow 暴露 UI 状态（推荐，比 LiveData 更现代）
    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUser(userId: String) {
        viewModelScope.launch {
            val user = repository.getUser(userId)
            _uiState.update { it.copy(user = user) }
        }
    }
}

// Activity 中获取（旋转后拿到的是同一个实例）
val viewModel: UserViewModel by viewModels()

// Compose 中获取
val viewModel: UserViewModel = viewModel()
```

**注意事项：**
- ViewModel 里不要持有 Activity/View/Context 的引用，会内存泄漏
- 需要 Context 用 `AndroidViewModel`（提供 Application Context）
- 推荐用 StateFlow/SharedFlow 替代 LiveData（Compose 生态更友好）

---

### 2. LiveData

**一句话定位：** 能感知生命周期的可观察数据，只在页面可见时通知更新。

**解决了什么问题：**
以前用 RxJava 或 EventBus 观察数据，页面销毁了还在收回调，轻则崩溃重则泄漏。LiveData 天生认识生命周期——页面不可见就不通知你，页面销毁了自动断开，再也不用手动取消订阅。

**核心原理：**

打个比方：LiveData 像一个智能快递员，他认识你的作息。你在家（STARTED/RESUMED）才给你送包裹，你不在家就不送但会记着，等你回来再把最新的包裹补上。你搬走了（DESTROYED）他就把你从配送名单里删掉。

技术层面：
- 内部维护一个 `SafeObserverMap`，每个观察者关联一个 `LifecycleOwner`
- 观察者的生命周期处于 `STARTED` 或 `RESUMED` 才算"活跃"
- 数据变化时只通知活跃观察者，非活跃的会在变活跃时收到最新值
- 生命周期到 `DESTROYED` 自动移除观察者
- `setValue()` 主线程用，`postValue()` 后台线程用（会合并多次 post）

**关键 API：**

| API | 用途 |
|-----|------|
| `MutableLiveData<T>` | 可变的 LiveData，用来发数据 |
| `LiveData<T>` | 只读的 LiveData，暴露给外部 |
| `observe(owner, observer)` | 绑定生命周期观察 |
| `Transformations.map()` | 数据转换（类似 RxJava 的 map） |
| `Transformations.switchMap()` | 触发式数据转换 |
| `MediatorLiveData` | 合并多个 LiveData 源 |

**代码示例：**

```kotlin
class MyViewModel : ViewModel() {
    // 私有可变，公开不可变——标准写法
    private val _userName = MutableLiveData<String>()
    val userName: LiveData<String> = _userName

    fun fetchUserName() {
        viewModelScope.launch {
            val name = api.getUserName()
            _userName.value = name  // 主线程直接 setValue
        }
    }
}

// Activity 中观察
viewModel.userName.observe(this) { name ->
    textView.text = name
}
```

**注意事项：**
- 新项目推荐 StateFlow 替代 LiveData（Flow 生态更强大）
- LiveData 的 `postValue` 有坑：连续调多次只收最后一次
- `observeForever` 必须手动 `removeObserver`，否则泄漏

---

### 3. Room

**一句话定位：** SQLite 的"高级封装"，写 SQL 不用写 JDBC 那套样板代码了。

**解决了什么问题：**
直接用 SQLite 要写一堆 `Cursor`、`getColumnIndex`、手动映射对象……又啰嗦又容易写错。Room 在 SQLite 上面包了一层，你只管定义数据类和接口，SQL 查询编译时就帮你检查对不对。

**核心原理：**

打个比方：Room 就像你雇了一个数据库管家。你只需要告诉他"我要一张用户表"（@Entity）、"帮我查所有用户"（@Dao 的 @Query），他就会帮你把建表 SQL、查询代码、对象映射全搞定。而且他在你下单之前就会检查你的要求合不合理（编译时验证），不会等你运行了才报错。

技术层面：
- 编译时通过 KSP 处理注解，自动生成 `RoomDatabase_Impl` 等实现类
- `@Entity` 类 → 自动生成建表 SQL
- `@Dao` 接口 → 自动生成 SQL 执行和 Cursor→对象映射代码
- `@Query` 里的 SQL 在编译时验证，表名写错了直接编译报错
- 内部封装了 `SQLiteOpenHelper`，管理数据库版本和连接
- 支持返回 `Flow<T>`，数据变化时自动通知

**关键 API：**

| API | 用途 |
|-----|------|
| `@Entity` | 标记数据类，对应一张表 |
| `@Dao` | 数据访问接口，写查询方法 |
| `@Database` | 数据库定义，列出所有表和版本号 |
| `@Query` / `@Insert` / `@Update` / `@Delete` | SQL 操作 |
| `@Relation` | 定义表关联 |
| `Room.databaseBuilder()` | 创建数据库实例 |
| `TypeConverter` | 自定义类型转换（如 Date↔Long） |

**代码示例：**

```kotlin
// 1. 定义实体（就是表结构）
@Entity(tableName = "users")
data class User(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    @ColumnInfo(name = "nickname") val name: String,
    val age: Int
)

// 2. 定义 DAO（数据访问接口）
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE age > :minAge")
    fun getUsersOlderThan(minAge: Int): Flow<List<User>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)
}

// 3. 定义数据库
@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

// 4. 使用
val db = Room.databaseBuilder(context, AppDatabase::class.java, "app.db").build()
db.userDao().getUsersOlderThan(18).collect { users ->
    // 数据库变化时自动触发
}
```

**注意事项：**
- 数据库操作默认在主线程会崩，记得用 `suspend` 或 `Flow`
- 升级数据库用 `Migration`，别用 `fallbackToDestructiveMigration()`（会丢数据）
- 大量数据用 `Paging` + Room 配合，别一次性全查出来

---

### 4. DataStore

**一句话定位：** SharedPreferences 的替代品，异步、不会卡主线程。

**解决了什么问题：**
SharedPreferences（SP）有几个老毛病：同步 IO 会卡主线程导致 ANR、没有类型安全、`apply()` 的原子性有问题。DataStore 用协程 + Flow 重新做了一遍，彻底解决了这些问题。

**核心原理：**

打个比方：SP 像一个小本子，每次读写都得等上一个人写完（apply 看似异步其实有坑）。DataStore 像一个自动存取柜——你把东西放进去（`updateData`），它保证原子性操作（要么全成功要么全失败），而且全程异步，绝不让你干等。

技术层面：
- **Preferences DataStore**：键值对存储，类似 SP 的用法，但用 Flow 暴露数据
- **Proto DataStore**：存自定义类型对象，需要定义 schema（Protocol Buffers 或自定义 Serializer）
- `updateData()` 内部是原子性的读-改-写事务，并发安全
- 数据存在文件里（XML 格式），通过协程异步读写
- 同一进程中对同一个文件只应创建一个 DataStore 实例（用委托保证单例）

**关键 API：**

| API | 用途 |
|-----|------|
| `preferencesDataStore(name)` | Kotlin 委托，创建 Preferences DataStore |
| `dataStore` | Kotlin 委托，创建 Proto DataStore |
| `dataStore.data` | Flow，监听数据变化 |
| `dataStore.edit { }` | 修改数据（原子操作） |
| `dataStore.updateData { }` | Proto DataStore 修改数据 |
| `intPreferencesKey()` | 定义键 |

**代码示例：**

```kotlin
// 创建（放在顶层或依赖注入）
val Context.settingsDataStore by preferencesDataStore(name = "settings")

// 定义键
object SettingsKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
    val USER_TOKEN = stringPreferencesKey("user_token")
}

// 读取
val darkModeFlow: Flow<Boolean> = context.settingsDataStore.data.map { prefs ->
    prefs[SettingsKeys.DARK_MODE] ?: false
}

// 写入
suspend fun setDarkMode(enabled: Boolean) {
    context.settingsDataStore.edit { prefs ->
        prefs[SettingsKeys.DARK_MODE] = enabled
    }
}
```

**注意事项：**
- 别在同一个进程创建多个 DataStore 实例访问同一文件，会出并发问题
- 简单配置用 Preferences DataStore，复杂对象用 Proto DataStore
- SP → DataStore 迁移有专门的 `SharedPreferencesMigration`，别手动搬数据

---

### 5. Navigation

**一句话定位：** 管理页面跳转的"总调度"，用图结构替代碎片化的事务管理。

**解决了什么问题：**
以前用 `FragmentManager.beginTransaction()` 管理页面跳转，代码又散又乱，传参靠 Bundle 容易写错 key，深链接更是噩梦。Navigation 把所有页面和跳转关系画成一张"地图"，跳转、传参、返回栈、深链接全部统一管理。

**核心原理：**

打个比方：Navigation 就像商场导航图。你站在商场入口（NavHost），想去三楼餐厅（Destination），导航图告诉你怎么走（NavController.navigate()），还能自动规划路线（深链接直达）。你走过的路它都记着（返回栈），随时可以原路返回。

技术层面：
- **NavGraph**：图结构，定义所有目的地（Destination）和它们之间的连线（Action）
- **NavController**：中央调度器，执行 navigate、popBackStack 等操作
- **NavHost**：UI 容器，根据当前目的地显示对应页面
- 支持 Fragment 和 Compose 两种宿主模式
- Safe Args 插件提供类型安全的参数传递（Fragment 模式）
- Compose 模式推荐用 `@Serializable` 数据类做路由，类型安全

**关键 API：**

| API | 用途 |
|-----|------|
| `NavHost(navController, startDestination)` | 导航容器 |
| `composable<T>()` | 注册 Compose 目的地 |
| `navController.navigate(route)` | 跳转 |
| `navController.popBackStack()` | 返回上一页 |
| `navController.previousBackStackEntry` | 获取上一页 |
| `navDeepLink()` | 定义深链接 |
| `NavigationSuiteScaffold` | 自适应导航栏（手机底部/平板侧边） |

**代码示例：**

```kotlin
// 定义路由（类型安全）
@Serializable data class Home(val tab: Int = 0)
@Serializable data class Detail(val itemId: String)

// 构建导航图
@Composable fun MyAppNavHost(navController: NavHostController) {
    NavHost(navController = navController, startDestination = Home()) {
        composable<Home> { backStackEntry ->
            val home = backStackEntry.toRoute<Home>()
            HomeScreen(onItemClick = { id ->
                navController.navigate(Detail(itemId = id))
            })
        }
        composable<Detail> { backStackEntry ->
            val detail = backStackEntry.toRoute<Detail>()
            DetailScreen(itemId = detail.itemId)
        }
    }
}
```

**注意事项：**
- Compose Navigation 目前不支持多返回栈（Multi-Back Stack）开箱即用，需要自己处理
- 避免在 ViewModel 里持有 NavController，会内存泄漏
- 深链接记得在 Manifest 里配 `nav-graph` 属性

---

### 6. Paging 3

**一句话定位：** 分页加载的"全家桶"，滚动到底自动加载下一页。

**解决了什么问题：**
加载大量数据（比如朋友圈、商品列表），不可能一次性全查出来。你得自己管理页码、处理加载状态、防止重复请求、做缓存……Paging 3 把这些全包了，你只需要告诉它"数据从哪来"。

**核心原理：**

打个比方：Paging 就像自动续杯的咖啡机。杯子（UI）快喝完了，咖啡机（PagingSource）自动再倒一杯。你不用管什么时候该续、续多少、咖啡机忙不忙，它自己全处理好了。

技术层面：
- **三层架构**：
  - `PagingSource`：数据源，定义怎么加载某一页（网络、数据库都行）
  - `Pager`：根据配置生成 `PagingData` 流，协调数据加载
  - `PagingDataAdapter` / `LazyPagingItems`：UI 层自动绑定和展示
- `RemoteMediator`：处理"网络 + 本地缓存"的分层数据源
- 内置内存缓存，已加载的页面不会重复请求
- 自动处理加载状态：Loading、Append、Prepend、Error
- `DiffUtil` 内置，只刷新变化的 Item

**关键 API：**

| API | 用途 |
|-----|------|
| `PagingSource<Key, Value>` | 定义数据源 |
| `RemoteMediator` | 网络+本地缓存的协调器 |
| `Pager(config) { source }` | 构建 PagingData 流 |
| `PagingConfig(pageSize)` | 配置页大小、预取距离 |
| `collectAsLazyPagingItems()` | Compose 中收集分页数据 |
| `PagingDataAdapter` | RecyclerView 适配器 |
| `loadState` | 观察加载状态 |

**代码示例：**

```kotlin
// 1. 定义数据源
class ArticlePagingSource(
    private val api: ArticleApi
) : PagingSource<Int, Article>() {
    override suspend fun load(params: LoadParams<Int>): LoadResult<Int, Article> {
        val page = params.key ?: 1
        return try {
            val articles = api.getArticles(page = page, size = params.loadSize)
            LoadResult.Page(
                data = articles,
                prevKey = if (page == 1) null else page - 1,
                nextKey = if (articles.isEmpty()) null else page + 1
            )
        } catch (e: Exception) {
            LoadResult.Error(e)
        }
    }
}

// 2. ViewModel 中构建
val pager = Pager(PagingConfig(pageSize = 20, prefetchDistance = 5)) {
    ArticlePagingSource(api)
}
val pagingData: Flow<PagingData<Article>> = pager.flow

// 3. Compose 中展示
val lazyItems = pagingData.collectAsLazyPagingItems()
LazyColumn {
    items(lazyItems.itemCount) { index ->
        ArticleCard(article = lazyItems[index]!!)
    }
    // 加载状态处理
    lazyItems.apply {
        when {
            loadState.refresh is LoadState.Loading -> { /* 首次加载中 */ }
            loadState.append is LoadState.Loading -> { /* 加载更多中 */ }
            loadState.refresh is LoadState.Error -> { /* 加载失败 */ }
        }
    }
}
```

**注意事项：**
- `pageSize` 建议设为 `prefetchDistance` 的 2~3 倍
- 网络分页用 `PagingSource`，网络+本地缓存用 `RemoteMediator`
- Paging 3 完全基于 Kotlin 协程和 Flow，不支持 RxJava

---

### 7. Hilt

**一句话定位：** Android 版的依赖注入工具，自动帮你把需要的对象"注入"到正确的地方。

**解决了什么问题：**
一个 Activity 可能需要 Repository、ViewModel、网络客户端……以前你得手动 new 出来一层层传递，或者写单例，代码耦合严重还不好测试。Hilt 让你声明"我需要什么"，它自动帮你创建和注入。

**核心原理：**

打个比方：Hilt 就像外卖平台。你（Activity）只需要在门口贴个单子说"我要一杯咖啡"（@Inject），外卖平台（Hilt）会自动找到能做咖啡的店（@Module），做好送到你手上。你不用管咖啡怎么做、用哪家店、送餐路线怎么走。

技术层面：
- 编译时通过 KSP 生成 Dagger 组件代码（不是运行时反射，性能好）
- `@HiltAndroidApp` 在 Application 上触发，生成应用级依赖容器（根组件）
- `@AndroidEntryPoint` 给每个 Android 类生成子容器，从父容器获取依赖
- 组件层级：`SingletonComponent` → `ActivityComponent` → `FragmentComponent` → `ViewComponent`
- `@Inject constructor` 告诉 Hilt "这个类可以自动创建"
- `@Module + @Provides` 告诉 Hilt "这个类的创建方式比较复杂，按我说的来"

**关键 API：**

| API | 用途 |
|-----|------|
| `@HiltAndroidApp` | 标记 Application，触发代码生成 |
| `@AndroidEntryPoint` | 标记需要注入的 Android 类 |
| `@HiltViewModel` | 标记 ViewModel |
| `@Inject constructor(...)` | 构造函数注入 |
| `@Module` + `@InstallIn` | 定义依赖提供模块 |
| `@Provides` | 提供实例 |
| `@Binds` | 接口绑定实现类 |
| `@Qualifier` | 区分同一类型的不同实例 |

**代码示例：**

```kotlin
// 1. Application
@HiltAndroidApp
class MyApp : Application()

// 2. 需要注入的类（构造函数注入）
class AnalyticsService @Inject constructor(
    private val retrofit: Retrofit
) {
    fun trackEvent(event: String) { /* ... */ }
}

// 3. 提供复杂依赖的模块
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit =
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
}

// 4. 在 Activity 中使用
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
    @Inject lateinit var analytics: AnalyticsService

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        analytics.trackEvent("page_view")
    }
}

// 5. 在 ViewModel 中使用
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository
) : ViewModel() { /* ... */ }
```

**注意事项：**
- ViewModel 必须用 `@HiltViewModel`，不能用 `@AndroidEntryPoint`
- `@Module` 的 `@InstallIn` 决定了作用域，别乱用 `SingletonComponent`
- 界面多的时候 Hilt 启动会慢（编译时生成代码多），可以考虑 Koin 做轻量替代

---

## 二、UI 类

### 8. Jetpack Compose

**一句话定位：** Android 的新一代 UI 框架，用 Kotlin 代码写界面，告别 XML。

**解决了什么问题：**
传统 View 体系用 XML 写布局、Java/Kotlin 写逻辑，状态一变就要 `findViewById` + 手动更新 UI，代码散落各处还容易出 bug。Compose 用声明式的方式——你描述 UI 长什么样，状态变了它自动帮你刷新该刷新的部分。

**核心原理：**

打个比方：传统 View 体系像手工作坊——状态变了你得自己拿锤子去敲对应的 UI 元素。Compose 像流水线——你给一个设计图纸（@Composable 函数），原材料（状态）一变，流水线自动重新生产受影响的部分，其他没变的直接复用。

技术层面：
- `@Composable` 函数描述 UI，Compose 编译器插件将其转换为高效的树结构
- **重组（Recomposition）**：状态变化时，Compose 只重新执行受影响的 Composable 函数，不会全量刷新
- **智能跳过（Skipping）**：如果 Composable 的参数没变，整个函数直接跳过
- 内部用 **Slot Table** 数据结构跟踪组合状态，高效定位需要更新的节点
- `remember` 跨重组保持值，`mutableStateOf` 创建可观察状态
- `Modifier` 链式 API 统一处理尺寸、间距、点击、滚动等所有 UI 属性

**关键 API：**

| API | 用途 |
|-----|------|
| `@Composable` | 标记可组合函数 |
| `mutableStateOf()` | 创建可观察状态 |
| `remember { }` | 跨重组保持值 |
| `derivedStateOf { }` | 派生状态，避免不必要的重组 |
| `LaunchedEffect` | 启动协程副作用 |
| `DisposableEffect` | 注册/清理副作用 |
| `CompositionLocal` | 隐式传递数据（类似 Context） |
| `Modifier` | 声明式 UI 修饰链 |

**代码示例：**

```kotlin
@Composable
fun UserListScreen(viewModel: UserViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // 只在第一次进入时加载
    LaunchedEffect(Unit) {
        viewModel.loadUsers()
    }

    when {
        uiState.isLoading -> CircularProgressIndicator()
        uiState.error != null -> Text("出错了: ${uiState.error}")
        else -> LazyColumn {
            items(uiState.users) { user ->
                UserCard(
                    user = user,
                    onClick = { viewModel.openDetail(user.id) },
                    modifier = Modifier.fillMaxWidth().padding(16.dp)
                )
            }
        }
    }
}
```

**注意事项：**
- Composable 函数不要有副作用（直接改外部变量、发网络请求等），用 `LaunchedEffect` 包
- 列表项用 `key { item.id }` 帮助 Compose 高效复用
- `remember` 不带 key 的话只在首次组合时执行，带 key 则 key 变化时重新执行
- 性能优化核心：减少重组范围 → 提取小 Composable、用 `remember`、用 `derivedStateOf`

---

### 9. Material 3

**一句话定位：** Google 的设计规范在 Compose 中的实现，一套代码搞定主题和组件。

**解决了什么问题：**
以前要统一 App 的配色、字体、圆角，得自己定义一堆 style 和 theme，维护起来很痛苦。Material 3 把这些全部标准化了——你只需要定义一套配色方案，所有组件自动适配。

**核心原理：**

打个比方：Material 3 就像一个装修公司的标准套餐。你选一个主色调（ColorScheme），它自动帮你把墙壁、家具、灯饰全部搭配好。Android 12+ 还支持"动态取色"——根据用户的壁纸自动生成配色，每换一张壁纸 App 换一套风格。

技术层面：
- `MaterialTheme` 是主题容器，提供三大子系统：
  - `ColorScheme`：配色方案（primary、secondary、surface、background 等）
  - `Typography`：排版系统（headlineLarge、bodyMedium、labelSmall 等）
  - `Shapes`：形状系统（卡片、按钮的圆角大小）
- Android 12+ 的 `dynamicLightColorScheme()` 从壁纸取色
- 所有 M3 组件（Button、Card、NavigationBar 等）内部读取 `MaterialTheme` 的值

**代码示例：**

```kotlin
@Composable
fun MyAppTheme(darkTheme: Boolean = isSystemInDarkTheme(), content: @Composable () -> Unit) {
    // Android 12+ 用动态取色，低版本用自定义配色
    val colorScheme = when {
        Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        }
        darkTheme -> darkColorScheme(
            primary = Color(0xFFBB86FC),
            secondary = Color(0xFF03DAC6)
        )
        else -> lightColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC6)
        )
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography(),  // 用默认排版，或自定义
        shapes = Shapes(),          // 用默认形状，或自定义
        content = content
    )
}

// 使用组件
@Composable
fun MyButton() {
    Button(onClick = { /* ... */ }) {
        Text("确定", style = MaterialTheme.typography.labelLarge)
    }
}
```

**注意事项：**
- 动态取色只支持 Android 12+，低版本需要自己定义 fallback 配色
- M3 的 `ExperimentalMaterial3Api` 很多组件还在实验阶段，升级大版本时注意 API 变化
- 自定义主题时别直接改组件颜色，改 `ColorScheme` 让组件自动适配

---

### 10. Compose Navigation

**一句话定位：** Navigation 组件的 Compose 版，在 Composable 函数之间跳转。

**核心原理：**

跟 Fragment 版的 Navigation 原理一样（NavGraph + NavController + NavHost），只是宿主从 Fragment 换成了 Composable 函数。推荐用 `@Serializable` 数据类定义路由，编译时类型检查，传参不会写错。

**代码示例：**

```kotlin
@Serializable data class Profile(val userId: String)
@Serializable data class Settings(val from: String)

@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(navController = navController, startDestination = Profile(userId = "me")) {
        composable<Profile> { entry ->
            val profile = entry.toRoute<Profile>()
            ProfileScreen(
                userId = profile.userId,
                onNavigateToSettings = { navController.navigate(Settings(from = "profile")) }
            )
        }
        composable<Settings> { entry ->
            val settings = entry.toRoute<Settings>()
            SettingsScreen(onBack = { navController.popBackStack() })
        }
    }
}
```

**注意事项：**
- 路由参数必须是可序列化的基本类型或 `@Serializable` 类
- 底部导航栏配合 `NavigationBar` + `NavigationSuiteScaffold` 使用
- 嵌套导航图用 `navigation(startDestination, route) { }` 包裹

---

### 11. ViewBinding

**一句话定位：** 自动为每个 XML 布局生成类型安全的绑定类，告别 `findViewById`。

**解决了什么问题：**
`findViewById` 返回 View，你得强转类型，ID 写错了运行时才崩。ViewBinding 编译时生成绑定类，每个有 ID 的 View 都是正确类型的属性，空指针和类型转换错误在编译时就能发现。

**核心原理：**

编译时 Gradle 插件扫描每个 XML 布局文件，根据文件名生成绑定类（`activity_main.xml` → `ActivityMainBinding`）。绑定类的 `inflate()` 内部调用 `LayoutInflater.inflate()`，然后对每个有 ID 的 View 做 `findViewById` 并缓存。本质上是帮你写了 `findViewById` 的样板代码。

**代码示例：**

```kotlin
// Activity 中
class MainActivity : AppCompatActivity() {
    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        // 直接用，类型安全，不会 NPE
        binding.titleText.text = "Hello"
        binding.submitButton.setOnClickListener { /* ... */ }
    }
}

// Fragment 中（注意在 onDestroyView 清理引用）
class MyFragment : Fragment(R.layout.fragment_my) {
    private var _binding: FragmentMyBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        _binding = FragmentMyBinding.bind(view)
        binding.textView.text = "Hello"
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null  // Fragment 的视图可能比 Fragment 本身先销毁
    }
}
```

**注意事项：**
- Fragment 里一定要在 `onDestroyView` 置 null，否则内存泄漏
- ViewBinding 比 DataBinding 轻量，不需要在 XML 里写 `<layout>` 和 `<data>`
- 新项目如果全用 Compose，就不需要 ViewBinding 了

---

### 12. Fragment

**一句话定位：** Activity 里的"可插拔模块"，有自己的生命周期和布局。

**核心原理：**

Fragment 不能独立存在，必须住在 Activity 里。它有自己的生命周期（比 Activity 多了 `onCreateView` 和 `onDestroyView`），视图可以独立创建和销毁。`FragmentManager` 管理所有 Fragment 的事务和返回栈。

**关键生命周期：**
```
onAttach → onCreate → onCreateView → onViewCreated →
onStart → onResume → onPause → onStop →
onDestroyView → onDestroy → onDetach
```

**代码示例：**

```kotlin
class ProfileFragment : Fragment(R.layout.fragment_profile) {
    private val viewModel: ProfileViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        val binding = FragmentProfileBinding.bind(view)

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.profile.collect { profile ->
                    binding.nameText.text = profile.name
                }
            }
        }
    }
}

// Activity 中切换 Fragment
supportFragmentManager.commit {
    setReorderingAllowed(true)
    replace<R.id.container>(ProfileFragment::class.java, args = bundleOf("id" to "123"))
    addToBackStack("profile")
}
```

**注意事项：**
- Fragment 里观察 LiveData/Flow 必须用 `viewLifecycleOwner`，别用 `this`（this 的生命周期比视图长）
- `setReorderingAllowed(true)` 可以优化事务执行顺序，推荐开启
- Fragment 间通信用共享 ViewModel 或 Result API，别直接互相引用

---

### 13. RecyclerView

**一句话定位：** 高效列表控件，通过复用 Item View 来显示大量数据。

**核心原理：**

打个比方：RecyclerView 就像自助餐厅。它只有固定数量的盘子（ViewHolder），吃完的盘子收回来洗干净给下一个人用（复用），而不是每个人发一个新盘子。这样不管来多少人，盘子数量始终可控。

技术层面：
- **ViewHolder 模式**：每个 Item 对应一个 ViewHolder，滚出屏幕的 ViewHolder 被回收复用
- **LayoutManager**：决定怎么排列（线性、网格、瀑布流）
- **DiffUtil**：用 Myers 差分算法计算新旧列表差异，只更新变化的 Item
- **ListAdapter**：内置 DiffUtil 的适配器，调用 `submitList()` 自动计算差异并更新

**代码示例：**

```kotlin
// 1. 定义 Adapter
class UserAdapter(
    private val onItemClick: (User) -> Unit
) : ListAdapter<User, UserViewHolder>(UserDiffCallback()) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder {
        val binding = ItemUserBinding.inflate(
            LayoutInflater.from(parent.context), parent, false
        )
        return UserViewHolder(binding, onItemClick)
    }

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
}

// 2. DiffUtil 回调
class UserDiffCallback : DiffUtil.ItemCallback<User>() {
    override fun areItemsTheSame(old: User, new: User) = old.id == new.id
    override fun areContentsTheSame(old: User, new: User) = old == new
}

// 3. ViewHolder
class UserViewHolder(
    private val binding: ItemUserBinding,
    private val onItemClick: (User) -> Unit
) : RecyclerView.ViewHolder(binding.root) {
    fun bind(user: User) {
        binding.nameText.text = user.name
        binding.root.setOnClickListener { onItemClick(user) }
    }
}

// 4. 使用
val adapter = UserAdapter { user -> showDetail(user) }
recyclerView.adapter = adapter
recyclerView.layoutManager = LinearLayoutManager(context)
adapter.submitList(userList)
```

**注意事项：**
- 别用 `notifyDataSetChanged()`，会全量刷新没有动画；用 `ListAdapter.submitList()`
- `submitList` 传同一个列表对象不会触发更新，需要传新对象
- Compose 项目用 `LazyColumn` 替代 RecyclerView

---

## 三、基础类 (Foundation)

### 14. WorkManager

**一句话定位：** 后台任务的"保险箱"，就算 App 被杀了、手机重启了，任务照样执行。

**解决了什么问题：**
后台任务（上传日志、同步数据）需要可靠执行，但 Android 的后台限制越来越严（Doze 模式、后台执行限制）。WorkManager 帮你选最合适的执行方式（JobScheduler / AlarmManager），保证任务一定能跑完。

**核心原理：**

打个比方：WorkManager 就像一个靠谱的快递公司。你把包裹（任务）交给它，它会在合适的时间送达。就算路上遇到暴风雨（App 被杀、手机重启），它也会在条件恢复后继续送。你还可以设定条件——"等有 WiFi 再送"（Constraints）。

技术层面：
- 内部用 SQLite 数据库持久化任务，设备重启后自动恢复
- API 23+ 用 `JobScheduler`，低版本用 `AlarmManager + BroadcastReceiver`
- 支持约束条件：网络类型、充电状态、电量、存储空间
- 支持重试策略（指数退避）、任务链（顺序/并行组合）
- 加急工作（Expedited Work）可以立即执行（类似 Foreground Service）

**代码示例：**

```kotlin
// 1. 定义 Worker
class UploadWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result {
        return try {
            uploadData()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }
}

// 2. 构建并调度
val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>()
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()
    )
    .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 10, TimeUnit.MINUTES)
    .addTag("upload")
    .build()

WorkManager.getInstance(context).enqueue(uploadWork)

// 3. 观察状态
WorkManager.getInstance(context).getWorkInfoByIdLiveData(uploadWork.id)
    .observe(this) { workInfo ->
        when (workInfo.state) {
            WorkInfo.State.SUCCEEDED -> showToast("上传成功")
            WorkInfo.State.FAILED -> showToast("上传失败")
            else -> {}
        }
    }

// 4. 链式调用（先压缩再上传）
WorkManager.getInstance(context)
    .beginWith(compressWork)
    .then(uploadWork)
    .enqueue()
```

**注意事项：**
- 不需要保证立即执行的任务用 WorkManager，需要立即执行的用 Foreground Service
- 周期任务最小间隔 15 分钟（系统限制）
- `CoroutineWorker` 里不要做超过 10 分钟的操作，会被系统杀掉

---

### 15. App Startup

**一句话定位：** 统一管理 App 启动时的初始化，替代每个库各自用 ContentProvider 的做法。

**核心原理：**

打个比方：以前每个库（WorkManager、Firebase 等）启动时都自己开一扇门（ContentProvider）进你的 App，门多了启动就慢。App Startup 把所有门合成一扇——一个 ContentProvider 统一按顺序初始化所有库。

技术层面：
- 每个库实现 `Initializer<T>` 接口，声明 `create()` 和 `dependencies()`
- App Startup 在 `Application.onCreate()` 之前通过 `InitializationProvider` 执行
- 按依赖关系拓扑排序，依次初始化
- 也可以禁用自动初始化，手动调用 `AppInitializer.getInstance().initializeComponent()`

**代码示例：**

```kotlin
// 1. 定义初始化器
class WorkManagerInitializer : Initializer<WorkManager> {
    override fun create(context: Context): WorkManager {
        val config = Configuration.Builder()
            .setMinimumLoggingLevel(Log.DEBUG)
            .build()
        WorkManager.initialize(context, config)
        return WorkManager.getInstance(context)
    }

    // 声明依赖（在 LoggerInitializer 之后初始化）
    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf(LoggerInitializer::class.java)
    }
}

// 2. 在 AndroidManifest.xml 中注册
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.example.WorkManagerInitializer"
        android:value="androidx.startup" />
</provider>
```

**注意事项：**
- 第三方库（如 WorkManager、Firebase）已经自带了 Startup 初始化器，不需要你手动写
- 如果想延迟初始化某个库，可以在 Manifest 里用 `tools:node="remove"` 禁用自动初始化
- Startup 只解决"启动时初始化"的问题，懒加载还得自己处理

---

### 16. SplashScreen

**一句话定位：** 统一的启动画面 API，Android 12+ 是系统行为，低版本兼容库帮你搞定。

**核心原理：**

Android 12 开始系统强制显示启动画面（带 App 图标和动画），你没法关掉只能自定义。`core-splashscreen` 兼容库把这个行为统一到 Android 6.0+。

启动画面分三个阶段：
1. **进入动画**（系统控制）：冷启动时系统显示
2. **等待阶段**（你控制）：App 在后台初始化，画面保持显示
3. **退出动画**（你控制）：初始化完成，画面消失

**代码示例：**

```kotlin
// 1. themes.xml 中定义主题（Android 12+ 自动生效）
<style name="Theme.App.Starting" parent="Theme.SplashScreen">
    <item name="windowSplashScreenBackground">@color/splash_background</item>
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_splash_animation</item>
    <item name="postSplashScreenTheme">@style/Theme.App</item>
</style>

// 2. AndroidManifest.xml 中使用
<activity
    android:name=".MainActivity"
    android:theme="@style/Theme.App.Starting"
    ... />

// 3. Activity 中控制
override fun onCreate(savedInstanceState: Bundle?) {
    val splashScreen = installSplashScreen()

    // 方式一：等数据准备好再关闭
    splashScreen.setKeepOnScreenCondition { !viewModel.isReady }

    // 方式二：自定义退出动画
    splashScreen.setOnExitAnimationListener { splashScreenView ->
        val fadeOut = ObjectAnimator.ofFloat(splashScreenView, View.ALPHA, 1f, 0f)
        fadeOut.duration = 300
        fadeOut.doOnEnd { splashScreenView.remove() }
        fadeOut.start()
    }

    super.onCreate(savedInstanceState)
}
```

**注意事项：**
- `installSplashScreen()` 必须在 `super.onCreate()` 之前调用
- 启动画面图标建议用 Vector Drawable，大小 240x240 dp
- Android 12+ 的启动画面动画时长由系统控制，你只能自定义退出动画

---

## 四、行为类 (Behavior)

### 17. Biometric

**一句话定位：** 一行代码弹出指纹/面部识别对话框，还能跟加密操作绑定。

**核心原理：**

Biometric 库封装了系统的 `BiometricPrompt`，不管底层是指纹、面部还是虹膜，API 都一样。还支持回退到设备凭据（PIN/密码/图案）。配合 Android Keystore，可以做到"验证了指纹才能解密密钥"。

**代码示例：**

```kotlin
// 1. 检查设备是否支持
val biometricManager = BiometricManager.from(context)
when (biometricManager.canAuthenticate(BiometricManager.Authenticators.BIOMETRIC_STRONG)) {
    BiometricManager.BIometricManager.BIOMETRIC_SUCCESS -> { /* 可以使用 */ }
    BiometricManager.BIOMETRIC_ERROR_NO_HARDWARE -> { /* 没有硬件 */ }
    BiometricManager.BIOMETRIC_ERROR_HW_UNAVAILABLE -> { /* 硬件不可用 */ }
}

// 2. 弹出验证对话框
val promptInfo = BiometricPrompt.PromptInfo.Builder()
    .setTitle("验证身份")
    .setSubtitle("请使用指纹或面部识别")
    .setNegativeButtonText("取消")
    .build()

val biometricPrompt = BiometricPrompt(
    this,                           // Activity 或 Fragment
    ContextCompat.getMainExecutor(this),
    object : BiometricPrompt.AuthenticationCallback() {
        override fun onAuthenticationSucceeded(result: BiometricPrompt.AuthenticationResult) {
            // 验证成功，可以执行敏感操作
            unlockSensitiveData()
        }
        override fun onAuthenticationError(errorCode: Int, errString: CharSequence) {
            // 错误处理
        }
    }
)
biometricPrompt.authenticate(promptInfo)
```

**注意事项：**
- `setNegativeButtonText` 和 `setAllowedAuthenticators(DEVICE_CREDENTIAL)` 不能同时用
- 生物识别失败次数过多会被系统锁定，需要等冷却时间
- 敏感数据用 Keystore + Biometric 绑定，别只靠生物识别做唯一防线

---

### 18. CameraX

**一句话定位：** 相机开发的一站式方案，几行代码搞定预览、拍照、录像。

**核心原理：**

CameraX 把相机功能拆成几个"用例"（Use Case），你需要什么就绑什么：
- **Preview**：显示相机预览画面
- **ImageCapture**：拍照
- **ImageAnalysis**：分析图像（如二维码扫描、ML Kit）
- **VideoCapture**：录像

CameraX 内部处理了设备兼容性问题（不同厂商的相机 API 行为不一致），你不用关心。

**代码示例：**

```kotlin
// 1. 获取相机提供者
val cameraProviderFuture = ProcessCameraProvider.getInstance(context)

cameraProviderFuture.addListener({
    val cameraProvider = cameraProviderFuture.get()

    // 2. 创建用例
    val preview = Preview.Builder().build().also {
        it.surfaceProvider = previewView.surfaceProvider
    }

    val imageCapture = ImageCapture.Builder()
        .setCaptureMode(ImageCapture.CAPTURE_MODE_MAXIMIZE_QUALITY)
        .build()

    // 3. 绑定到生命周期
    cameraProvider.unbindAll()  // 先解绑旧的
    cameraProvider.bindToLifecycle(
        this,                      // 生命周期所有者
        CameraSelector.DEFAULT_BACK_CAMERA,
        preview,
        imageCapture
    )
}, ContextCompat.getMainExecutor(context))

// 4. 拍照
imageCapture.takePicture(
    ImageCapture.OutputFileOptions.Builder(photoFile).build(),
    ContextCompat.getMainExecutor(context),
    object : ImageCapture.OnImageSavedCallback {
        override fun onImageSaved(output: ImageCapture.OutputFileResults) { /* 成功 */ }
        override fun onError(exc: ImageCaptureException) { /* 失败 */ }
    }
)
```

**注意事项：**
- `bindToLifecycle` 绑定生命周期，Activity 暂停时相机自动释放，恢复时自动打开
- 同时绑定多个用例时，CameraX 自动协调分辨率和帧率
- Compose 中用 `AndroidView { PreviewView(it) }` 嵌入预览

---

### 19. Media3

**一句话定位：** 音视频播放的"瑞士军刀"，一个库搞定播放、后台服务、通知栏控制。

**核心原理：**

Media3 把以前分散的 ExoPlayer、MediaSession、MediaCompat 合并成一个统一的库。核心是 `Player` 接口（定义播放能力），`ExoPlayer` 是默认实现。`MediaSession` 把播放状态广播给系统（锁屏控制、蓝牙控制），`MediaController` 远程控制播放。

打个比方：`ExoPlayer` 是播放器本身，`MediaSession` 是遥控器信号发射器（告诉系统"我在放什么歌"），`MediaController` 是遥控器（别的 App 或系统 UI 通过它控制你）。

**代码示例：**

```kotlin
// 1. 创建播放器
val player = ExoPlayer.Builder(context).build()

// 2. 设置数据源
player.setMediaItem(MediaItem.fromUri("https://example.com/video.mp4"))
// 或播放列表
player.setMediaItems(listOf(
    MediaItem.fromUri("https://example.com/song1.mp3"),
    MediaItem.fromUri("https://example.com/song2.mp3")
))
player.prepare()
player.playWhenReady = true

// 3. 视频播放 UI
PlayerView(context).apply { player = this@apply.player }

// 4. 后台播放 + 通知栏控制
class PlaybackService : MediaSessionService() {
    private var mediaSession: MediaSession? = null

    override fun onCreate() {
        super.onCreate()
        val player = ExoPlayer.Builder(this).build()
        mediaSession = MediaSession.Builder(this, player)
            .setCallback(object : MediaSession.Callback {
                override fun onAddMediaItems(
                    session: MediaSession,
                    controller: MediaSession.ControllerInfo,
                    mediaItems: List<MediaItem>
                ): ListenableFuture<List<MediaItem>> {
                    // 处理播放请求
                    return Futures.immediateFuture(mediaItems)
                }
            })
            .build()
    }

    override fun onGetSession(controllerInfo: MediaSession.ControllerInfo) = mediaSession
}
```

**注意事项：**
- `PlayerView` 在 Compose 中用 `AndroidView { PlayerView(it) }` 嵌入
- 后台播放必须在 Manifest 声明 `MediaSessionService` 和前台通知权限
- Media3 完全替代了旧版 ExoPlayer 和 MediaCompat，新项目直接用 Media3

---

### 20. Notifications

**一句话定位：** 在 App 界面之外给用户发消息，状态栏、锁屏、通知栏都能看到。

**核心原理：**

通知通过 `NotificationManager` 系统服务发布。Android 8.0+ 要求所有通知必须分配到**通知渠道**（NotificationChannel），用户可以对每个渠道独立控制（开关、声音、是否弹出）。通知的重要性级别决定它有多"打扰"用户。

**代码示例：**

```kotlin
// 1. 创建通知渠道（Android 8.0+，只需创建一次）
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    val channel = NotificationChannel(
        CHANNEL_ID,           // 渠道 ID
        "消息通知",            // 用户看到的名称
        NotificationManager.IMPORTANCE_DEFAULT  // 重要程度
    ).apply {
        description = "接收聊天消息"
        enableVibration(true)
    }
    notificationManager.createNotificationChannel(channel)
}

// 2. 构建通知
val notification = NotificationCompat.Builder(context, CHANNEL_ID)
    .setSmallIcon(R.drawable.ic_notification)       // 必须设置
    .setContentTitle("新消息")
    .setContentText("你收到了一条新消息")
    .setLargeIcon(BitmapFactory.decodeResource(resources, R.drawable.avatar))
    .setPriority(NotificationCompat.PRIORITY_DEFAULT)
    .setAutoCancel(true)                            // 点击后自动消失
    .setContentIntent(pendingIntent)                // 点击跳转
    .addAction(R.drawable.ic_reply, "回复", replyPendingIntent)  // 操作按钮
    .build()

// 3. 发送
NotificationManagerCompat.from(context).notify(NOTIFICATION_ID, notification)
```

**注意事项：**
- `setSmallIcon` 是必须的，不设的话通知不显示
- Android 13+ 需要运行时请求通知权限 `POST_NOTIFICATIONS`
- 通知渠道一旦创建并提交给系统，后续修改只有部分属性生效（名称、描述可以改，重要性不能改）
- 大图通知用 `setStyle(NotificationCompat.BigPictureStyle())`
- 进度条通知用 `setProgress(max, current, indeterminate)`

---

## 速查总表

| # | 库名 | 分类 | 一句话定位 | 核心原理 |
|---|------|------|-----------|---------|
| 1 | **ViewModel** | 架构 | 屏幕级状态容器，旋转不丢数据 | ViewModelStore 缓存实例，生命周期比 Activity 长 |
| 2 | **LiveData** | 架构 | 感知生命周期的可观察数据 | 关联 LifecycleOwner，活跃时才通知，销毁时自动断开 |
| 3 | **Room** | 架构 | SQLite 的 ORM 封装 | 编译时注解处理生成 SQL 代码，编译时验证查询 |
| 4 | **DataStore** | 架构 | SP 的替代品，异步存储 | 协程 + Flow，原子性读写，文件持久化 |
| 5 | **Navigation** | 架构 | 页面跳转统一管理 | 图结构管理目的地，NavController 中央调度 |
| 6 | **Paging 3** | 架构 | 分页加载全家桶 | 三层架构（Source→Pager→Adapter），自动加载和缓存 |
| 7 | **Hilt** | 架构 | 依赖注入框架 | 编译时生成 Dagger 组件，按层级自动注入 |
| 8 | **Compose** | UI | 声明式 UI 框架 | @Composable 函数 + 智能重组 + Slot Table |
| 9 | **Material 3** | UI | 设计规范实现 | ColorScheme + Typography + Shapes 三大子系统 |
| 10 | **Compose Nav** | UI | Compose 页面导航 | Navigation 的 Compose 版，@Serializable 类型安全路由 |
| 11 | **ViewBinding** | UI | XML 布局类型安全绑定 | 编译时生成绑定类，替代 findViewById |
| 12 | **Fragment** | UI | 可插拔的 UI 模块 | 独立生命周期，FragmentManager 管理事务 |
| 13 | **RecyclerView** | UI | 高效列表控件 | ViewHolder 复用 + DiffUtil 差异更新 |
| 14 | **WorkManager** | 基础 | 可靠的后台任务调度 | SQLite 持久化 + JobScheduler/AlarmManager 自适应 |
| 15 | **Startup** | 基础 | 启动初始化统一管理 | 共享 ContentProvider + Initializer 依赖排序 |
| 16 | **SplashScreen** | 基础 | 统一启动画面 | 兼容 Android 12+ 系统行为，三阶段动画 |
| 17 | **Biometric** | 行为 | 指纹/面部识别 | 封装 BiometricPrompt，支持 Keystore 加密绑定 |
| 18 | **CameraX** | 行为 | 相机开发一站式方案 | Use Case 架构，自动处理设备兼容性 |
| 19 | **Media3** | 行为 | 音视频播放统一库 | Player 接口 + MediaSession + MediaController |
| 20 | **Notifications** | 行为 | 系统通知 | NotificationManager + Channel + NotificationCompat |

---

> 📖 所有内容基于 [developer.android.com/jetpack](https://developer.android.com/jetpack) 官方文档整理，如有出入以官方文档为准。
