# Android Jetpack 核心库速查手册（Java 版）

> 面向有经验的 Android 开发者 | 基于 developer.android.com 官方文档整理
> 所有代码示例均为 Java，适合 Java 技术栈团队参考

---

## 目录

- [一、架构类（Architecture）](#一架构类architecture)
  - [1. ViewModel](#1-viewmodel)
  - [2. LiveData](#2-livedata)
  - [3. Room](#3-room)
  - [4. DataStore](#4-datastore)
  - [5. Navigation](#5-navigation)
  - [6. Paging 3](#6-paging-3)
  - [7. Hilt（Dagger Hilt）](#7-hiltdagger-hilt)
- [二、UI 类](#二ui-类)
  - [8. ViewBinding](#8-viewbinding)
  - [9. Fragment](#9-fragment)
  - [10. RecyclerView（ListAdapter + DiffUtil）](#10-recyclerviewlistadapter--diffutil)
- [三、基础类（Foundation）](#三基础类foundation)
  - [11. WorkManager](#11-workmanager)
  - [12. App Startup](#12-app-startup)
  - [13. SplashScreen](#13-splashscreen)
- [速查总表](#速查总表)

---

## 一、架构类（Architecture）

---

### 1. ViewModel

#### 一句话定位

屏幕级的状态容器，旋转屏幕数据不丢。

#### 解决了什么问题

Activity 旋转一下就被销毁重建，之前的数据全没了。以前你得自己 `onSaveInstanceState` 手动存取，又麻烦又容易漏。ViewModel 就是来干这个的——它活得比 Activity 长，旋转时数据自动保留。

#### 适用场景

- 页面需要展示从网络或数据库加载的数据，旋转后不想重新请求
- 多个 Fragment 共享同一份数据（通过 Activity 级别的 ViewModel）
- 需要持有一些跟 UI 交互相关的临时状态（如列表选中项、表单输入内容）
- 需要执行一些跟 UI 生命周期绑定的异步任务（网络请求、数据库操作）

#### 核心原理

打个比方：Activity 是工位，ViewModel 是你工位上锁着的抽屉。工位被拆了重建（旋转），但抽屉还在，新工位打开同一个抽屉，东西一样不少。

技术层面：

- ViewModel 的生命周期绑定到 `ViewModelStoreOwner`（Activity、Fragment、NavBackStackEntry），不是绑定到 Activity 本身
- 内部通过 `ViewModelStore`（本质是一个 HashMap）存所有 ViewModel 实例，Key 是类的全限定名
- 旋转时 Activity 重建，但 `ViewModelStore` 通过 `NonConfigurationInstances`（Activity 的一个成员变量）保留下来，新 Activity 拿到的是同一个 ViewModelStore
- Activity 真正 finish 时才调 `onCleared()` 清理，此时可以取消网络请求、关闭数据库连接等
- `SavedStateHandle` 解决进程被杀后的数据恢复问题——ViewModelStore 只能扛住配置变更，扛不住进程死亡；SavedStateHandle 把数据存到 Bundle 里，进程重建后自动恢复

**生命周期对比：**

```
Activity:    onCreate → onResume → (旋转) → onDestroy → onCreate → onResume → (finish) → onDestroy
ViewModel:   onCreate → onResume → (旋转，不受影响) → onResume → (finish) → onCleared
```

#### 关键 API

| API | 用途 | 说明 |
|-----|------|------|
| `ViewModel` | 基类 | 放状态和业务逻辑，不要放 View 引用 |
| `AndroidViewModel` | 带 Context 的基类 | 需要 Application Context 时用，内部持有 Application |
| `ViewModelProvider` | 创建/获取 ViewModel | 通过 Factory 模式创建实例 |
| `ViewModelProvider.Factory` | 工厂接口 | ViewModel 有带参构造函数时必须自定义 Factory |
| `SavedStateHandle` | 进程重建数据恢复 | 自动保存/恢复 Bundle 数据 |
| `ViewModelProvider.AndroidViewModelFactory` | 默认工厂 | 支持无参和 AndroidViewModel |
| `new ViewModelProvider(owner, factory).get(cls)` | 获取实例 | Java 中获取 ViewModel 的标准写法 |

#### Gradle 依赖

```groovy
dependencies {
    implementation "androidx.lifecycle:lifecycle-viewmodel:2.8.7"
    implementation "androidx.lifecycle:lifecycle-viewmodel-savedstate:2.8.7"
}
```

#### 代码示例

**基础用法：**

```java
// 1. 定义 ViewModel
public class UserViewModel extends ViewModel {

    private final MutableLiveData<User> userLiveData = new MutableLiveData<>();
    private final UserRepository repository;

    // 无参构造——ViewModelProvider.AndroidViewModelFactory 可以自动创建
    public UserViewModel() {
        repository = new UserRepository();
    }

    public LiveData<User> getUserLiveData() {
        return userLiveData;
    }

    public void loadUser(String userId) {
        // 注意：这里简化了线程切换，实际项目应该用 Executor 或 RxJava
        repository.getUser(userId, new Callback<User>() {
            @Override
            public void onSuccess(User user) {
                userLiveData.setValue(user);
            }

            @Override
            public void onError(Throwable t) {
                // 错误处理
            }
        });
    }

    @Override
    protected void onCleared() {
        // ViewModel 被销毁时清理资源
        repository.cleanup();
        super.onCleared();
    }
}

// 2. 在 Activity 中获取（旋转后拿到的是同一个实例）
public class UserActivity extends AppCompatActivity {

    private UserViewModel viewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_user);

        viewModel = new ViewModelProvider(this).get(UserViewModel.class);

        viewModel.getUserLiveData().observe(this, user -> {
            // 更新 UI
            nameTextView.setText(user.getName());
            ageTextView.setText(String.valueOf(user.getAge()));
        });

        viewModel.loadUser("123");
    }
}
```

**带参数的 ViewModel（自定义 Factory）：**

```java
// 1. 定义带参 ViewModel
public class DetailViewModel extends ViewModel {

    private final String itemId;
    private final MutableLiveData<ItemDetail> detailLiveData = new MutableLiveData<>();
    private final ItemRepository repository;

    public DetailViewModel(String itemId, ItemRepository repository) {
        this.itemId = itemId;
        this.repository = repository;
    }

    // ...
}

// 2. 自定义 Factory
public class DetailViewModelFactory implements ViewModelProvider.Factory {

    private final String itemId;
    private final ItemRepository repository;

    public DetailViewModelFactory(String itemId, ItemRepository repository) {
        this.itemId = itemId;
        this.repository = repository;
    }

    @NonNull
    @Override
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        if (modelClass.isAssignableFrom(DetailViewModel.class)) {
            return (T) new DetailViewModel(itemId, repository);
        }
        throw new IllegalArgumentException("Unknown ViewModel class");
    }
}

// 3. 在 Activity 中使用
DetailViewModelFactory factory = new DetailViewModelFactory(itemId, repository);
DetailViewModel viewModel = new ViewModelProvider(this, factory).get(DetailViewModel.class);
```

**使用 SavedStateHandle（进程重建后恢复数据）：**

```java
public class SearchViewModel extends ViewModel {

    private final SavedStateHandle savedStateHandle;
    private static final String KEY_QUERY = "search_query";

    public SearchViewModel(SavedStateHandle savedStateHandle) {
        this.savedStateHandle = savedStateHandle;
    }

    // 从 SavedStateHandle 读取（进程重建后也能恢复）
    public LiveData<String> getSearchQuery() {
        return savedStateHandle.getLiveData(KEY_QUERY, ""); // 第二个参数是默认值
    }

    public void setSearchQuery(String query) {
        savedStateHandle.set(KEY_QUERY, query);
    }
}

// 使用 SavedStateHandle 需要用 AbstractSavedStateViewModelFactory
public class SearchViewModelFactory extends AbstractSavedStateViewModelFactory {

    @NonNull
    @Override
    protected <T extends ViewModel> T create(
            @NonNull String key, @NonNull Class<T> modelClass,
            @NonNull SavedStateHandle handle) {
        if (modelClass.isAssignableFrom(SearchViewModel.class)) {
            return (T) new SearchViewModel(handle);
        }
        throw new IllegalArgumentException("Unknown ViewModel class");
    }
}
```

**Fragment 间共享 ViewModel：**

```java
// 两个 Fragment 共享数据——传入 activity 作为 ViewModelStoreOwner
public class ListFragment extends Fragment {

    private SharedViewModel sharedViewModel;

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        // 关键：用 requireActivity() 而不是 this
        sharedViewModel = new ViewModelProvider(requireActivity()).get(SharedViewModel.class);
    }
}

public class DetailFragment extends Fragment {

    private SharedViewModel sharedViewModel;

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        // 拿到的是同一个实例，因为都是 Activity 级别
        sharedViewModel = new ViewModelProvider(requireActivity()).get(SharedViewModel.class);
        sharedViewModel.getSelectedItem().observe(getViewLifecycleOwner(), item -> {
            // 更新详情 UI
        });
    }
}
```

#### 最佳实践

- ViewModel 里**不要**持有 Activity、View、Context（非 Application）的引用，会导致内存泄漏
- 需要 Context 时用 `AndroidViewModel`，它提供的是 Application Context，不会泄漏
- ViewModel 只管数据和业务逻辑，UI 更新通过 LiveData/Flow 通知，不要在 ViewModel 里操作 View
- 网络请求和数据库操作放在 Repository 层，ViewModel 只调用 Repository
- `onCleared()` 里取消所有进行中的异步操作

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 旋转后 ViewModel 数据没了 | 用了 `this` 而不是 Activity 作为 ViewModelStoreOwner | Fragment 中用 `requireActivity()` |
| 内存泄漏 | ViewModel 持有了 View 或 Activity 引用 | 只持有 Application Context 或无 Context |
| 进程被杀后数据丢失 | ViewModelStore 只扛配置变更，扛不住进程死亡 | 用 SavedStateHandle |
| ViewModel 有带参构造函数报错 | 默认 Factory 只支持无参构造 | 自定义 ViewModelProvider.Factory |

---

### 2. LiveData

#### 一句话定位

能感知生命周期的可观察数据，只在页面可见时通知更新。

#### 解决了什么问题

以前用 RxJava 或 EventBus 观察数据，页面销毁了还在收回调，轻则崩溃重则泄漏。LiveData 天生认识生命周期——页面不可见就不通知你，页面销毁了自动断开，再也不用手动取消订阅。

#### 适用场景

- ViewModel 向 Activity/Fragment 传递数据更新
- Fragment 间通信（通过共享 ViewModel 中的 LiveData）
- 需要生命周期感知的数据观察（只在页面可见时更新 UI）
- 数据库变化监听（Room 的 DAO 方法可以直接返回 LiveData）

#### 核心原理

打个比方：LiveData 像一个智能快递员，他认识你的作息。你在家（STARTED/RESUMED）才给你送包裹，你不在家就不送但会记着，等你回来再把最新的包裹补上。你搬走了（DESTROYED）他就把你从配送名单里删掉。

技术层面：

- 内部维护一个 `SafeObserverMap`，每个观察者关联一个 `LifecycleOwner`
- 观察者的生命周期处于 `STARTED` 或 `RESUMED` 才算"活跃"（Active）
- 数据变化时只通知活跃观察者，非活跃的会在变活跃时收到最新值（"粘性"特性）
- 生命周期到 `DESTROYED` 自动移除观察者，不会泄漏
- `setValue()` 只能在主线程调用，`postValue()` 可以在后台线程调用（但会合并多次 post，只保留最后一次值）
- 版本号机制：每次 `setValue` 版本号 +1，观察者记录自己上次收到的版本号，只分发版本号更新的数据

**数据分发流程：**

```
setValue("新数据")
  → version++
  → 遍历所有观察者
    → 判断 Lifecycle 状态是否 >= STARTED
      → 是：比较版本号，有更新就分发
      → 否：标记为"需要更新"，等变活跃时再分发
    → 判断 Lifecycle 状态是否 == DESTROYED
      → 是：自动移除观察者
```

#### 关键 API

| API | 用途 | 说明 |
|-----|------|------|
| `MutableLiveData<T>` | 可变的 LiveData | 用来发数据，只暴露给内部 |
| `LiveData<T>` | 只读的 LiveData | 暴露给外部，防止外部修改 |
| `observe(owner, observer)` | 绑定生命周期观察 | 最常用的方式，自动管理生命周期 |
| `observeForever(observer)` | 永远观察 | 必须手动 removeObserver，否则泄漏 |
| `setValue(value)` | 主线程更新数据 | 只能在主线程调用 |
| `postValue(value)` | 后台线程更新数据 | 可以在任意线程调用 |
| `getValue()` | 获取当前值 | 可以在任意线程调用 |
| `Transformations.map()` | 数据转换 | 类似 RxJava 的 map，返回新的 LiveData |
| `Transformations.switchMap()` | 触发式数据转换 | 输入变化时切换到新的 LiveData 源 |
| `MediatorLiveData` | 合并多个 LiveData | 可以监听多个 LiveData 源并合并结果 |

#### Gradle 依赖

```groovy
dependencies {
    implementation "androidx.lifecycle:lifecycle-livedata:2.8.7"
    implementation "androidx.lifecycle:lifecycle-livedata-core:2.8.7"
}
```

#### 代码示例

**基础用法：**

```java
public class MyViewModel extends ViewModel {

    // 标准写法：私有 MutableLiveData，公开 LiveData
    private final MutableLiveData<String> userName = new MutableLiveData<>();
    private final MutableLiveData<Boolean> isLoading = new MutableLiveData<>(false);
    private final MutableLiveData<String> errorMessage = new MutableLiveData<>();

    public LiveData<String> getUserName() { return userName; }
    public LiveData<Boolean> getIsLoading() { return isLoading; }
    public LiveData<String> getErrorMessage() { return errorMessage; }

    public void fetchUserName() {
        isLoading.setValue(true);
        repository.getUserName(new Callback<String>() {
            @Override
            public void onSuccess(String name) {
                isLoading.setValue(false);
                userName.setValue(name);  // 主线程用 setValue
            }

            @Override
            public void onError(Throwable t) {
                isLoading.setValue(false);
                errorMessage.setValue(t.getMessage());
            }
        });
    }
}

// Activity 中观察
public class MyActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        MyViewModel viewModel = new ViewModelProvider(this).get(MyViewModel.class);

        viewModel.getUserName().observe(this, name -> {
            textView.setText(name);
        });

        viewModel.getIsLoading().observe(this, loading -> {
            progressBar.setVisibility(loading ? View.VISIBLE : View.GONE);
        });

        viewModel.getErrorMessage().observe(this, error -> {
            if (error != null) {
                Toast.makeText(this, error, Toast.LENGTH_SHORT).show();
            }
        });
    }
}
```

**Transformations.map（数据转换）：**

```java
public class UserViewModel extends ViewModel {

    private final LiveData<User> userLiveData;
    // map 转换：User → 格式化的用户名
    private final LiveData<String> formattedName;

    public UserViewModel(UserRepository repository) {
        userLiveData = repository.getUserLiveData();
        formattedName = Transformations.map(userLiveData, user -> {
            if (user == null) return "未知用户";
            return user.getLastName() + ", " + user.getFirstName();
        });
    }

    public LiveData<String> getFormattedName() { return formattedName; }
}
```

**Transformations.switchMap（触发式数据切换）：**

```java
// 经典场景：根据 userId 切换不同的 LiveData 源
public class DetailViewModel extends ViewModel {

    private final MutableLiveData<String> userIdLiveData = new MutableLiveData<>();
    // switchMap：userId 变了，自动切换到对应的 LiveData 源
    private final LiveData<ItemDetail> detailLiveData;

    public DetailViewModel(ItemRepository repository) {
        detailLiveData = Transformations.switchMap(userIdLiveData, userId -> {
            if (userId == null || userId.isEmpty()) {
                return AbsentLiveData.create(); // 返回空的 LiveData
            }
            return repository.loadDetail(userId); // 返回新的 LiveData
        });
    }

    public void setUserId(String userId) {
        userIdLiveData.setValue(userId);
    }

    public LiveData<ItemDetail> getDetailLiveData() { return detailLiveData; }
}

// AbsentLiveData：一个永远不会发数据的空 LiveData
public class AbsentLiveData extends LiveData<Void> {
    private AbsentLiveData() {}
    public static <T> LiveData<T> create() {
        return new AbsentLiveData();
    }
}
```

**MediatorLiveData（合并多个数据源）：**

```java
// 场景：同时监听本地缓存和网络数据，任一变化都触发更新
public class CombinedLiveData extends MediatorLiveData<UserProfile> {

    public CombinedLiveData(LiveData<User> userLiveData, LiveData<List<Order>> ordersLiveData) {
        // 添加源1
        addSource(userLiveData, user -> {
            setValue(merge(user, ordersLiveData.getValue()));
        });
        // 添加源2
        addSource(ordersLiveData, orders -> {
            setValue(merge(userLiveData.getValue(), orders));
        });
    }

    private UserProfile merge(User user, List<Order> orders) {
        return new UserProfile(user, orders);
    }
}
```

**自定义 LiveData（监听数据源）：**

```java
// 场景：监听位置变化
public class LocationLiveData extends LiveData<Location> {

    private final LocationManager locationManager;
    private final LocationListener listener = new LocationListener() {
        @Override
        public void onLocationChanged(Location location) {
            setValue(location); // 有新位置就更新
        }
        // ... 其他回调
    };

    public LocationLiveData(Context context) {
        locationManager = (LocationManager) context.getSystemService(Context.LOCATION_SERVICE);
    }

    // 第一个观察者注册时开始监听
    @Override
    protected void onActive() {
        locationManager.requestLocationUpdates(
            LocationManager.GPS_PROVIDER, 0, 0, listener);
    }

    // 最后一个观察者移除时停止监听（省电）
    @Override
    protected void onInactive() {
        locationManager.removeUpdates(listener);
    }
}
```

#### 最佳实践

- 永远用"私有 MutableLiveData + 公开 LiveData"的模式，防止外部篡改数据
- Fragment 中观察 LiveData 必须用 `getViewLifecycleOwner()`，不能用 `this`（Fragment 的生命周期比视图长，用 this 会导致重复观察或崩溃）
- `postValue` 有坑：连续调多次只保留最后一次值，且不保证立即分发。如果对数据一致性要求高，确保在主线程用 `setValue`
- 需要合并、转换数据时优先用 `Transformations` 和 `MediatorLiveData`，别在观察者回调里手动处理

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| Fragment 中收到多次回调 | 用了 `this` 而不是 `getViewLifecycleOwner()` | 改用 `getViewLifecycleOwner()` |
| postValue 丢了数据 | 连续 post 只保留最后一次 | 主线程用 setValue |
| observeForever 泄漏 | 没有手动 removeObserver | 在 onDestroy 中 removeObserver |
| 收到旧数据 | LiveData 的"粘性"特性 | 用 SingleLiveEvent 包装（事件型数据） |

**SingleLiveEvent 实现（一次性事件，不会重复接收）：**

```java
public class SingleLiveEvent<T> extends MutableLiveData<T> {

    private final AtomicBoolean pending = new AtomicBoolean(false);

    @MainThread
    public void call(T value) {
        pending.set(true);
        setValue(value);
    }

    @MainThread
    @Override
    public void setValue(T value) {
        super.setValue(value);
    }

    @Override
    public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
        super.observe(owner, new Observer<T>() {
            @Override
            public void onChanged(T t) {
                if (pending.compareAndSet(true, false)) {
                    observer.onChanged(t);
                }
            }
        });
    }
}
```

---

### 3. Room

#### 一句话定位

SQLite 的"高级封装"，编译时帮你检查 SQL 对不对，不用手写 Cursor 那套样板代码了。

#### 解决了什么问题

直接用 SQLite 要写一堆 `SQLiteDatabase`、`Cursor`、`getColumnIndex`、手动映射对象……又啰嗦又容易写错（SQL 写错了运行时才崩）。Room 在 SQLite 上面包了一层 ORM，你只管定义实体类和接口，SQL 在编译时就帮你检查。

#### 适用场景

- 本地数据持久化（用户信息、设置、离线缓存）
- 需要结构化存储的复杂数据（关系型数据、多表关联）
- 需要监听数据变化并自动更新 UI（配合 LiveData）
- 大量数据的增删改查（Room 的性能接近原生 SQLite）

#### 核心原理

打个比方：Room 就像你雇了一个数据库管家。你只需要告诉他"我要一张用户表"（@Entity）、"帮我查所有成年用户"（@Dao 的 @Query），他就会帮你把建表 SQL、查询代码、对象映射全搞定。而且他在你下单之前就会检查你的要求合不合理（编译时验证），不会等你运行了才报错。

技术层面：

- 编译时通过 KSP / AnnotationProcessor 处理注解，自动生成 `RoomDatabase_Impl`、`Dao_Impl` 等实现类
- `@Entity` 类 → 自动生成 `CREATE TABLE IF NOT EXISTS` SQL，字段名默认用成员变量名，可用 `@ColumnInfo` 自定义
- `@Dao` 接口 → 自动生成 SQL 执行代码和 `Cursor → 对象` 映射代码
- `@Query` 里的 SQL 在编译时做语法和表/列名验证，写错了直接编译报错
- `@ForeignKey` 定义外键约束，`@Index` 定义索引
- 内部封装了 `SQLiteOpenHelper`，管理数据库版本和连接
- 支持返回 `LiveData<T>`（数据变化时自动发通知）、`Flow<T>`（Kotlin）、`DataSource.Factory`（Paging）
- 数据库操作默认不允许在主线程执行，会直接抛异常

**Room 架构图：**

```
┌─────────────┐     ┌─────────────┐     ┌──────────────────┐
│   Entity     │────▶│    DAO      │────▶│  RoomDatabase    │
│ (数据表结构)  │     │ (数据访问接口)│     │  (数据库实例)      │
└─────────────┘     └─────────────┘     └──────────────────┘
      │                    │                      │
  @Entity             @Query/@Insert          Room.databaseBuilder()
  @PrimaryKey         @Update/@Delete         .build()
  @ColumnInfo         @Relation
  @ForeignKey         @TypeConverter
```

#### 关键 API

| API | 用途 | 说明 |
|-----|------|------|
| `@Entity` | 标记数据类 | 对应一张数据库表 |
| `@PrimaryKey` | 主键 | 支持 `autoGenerate = true` 自增 |
| `@ColumnInfo` | 列名映射 | 自定义列名、默认值等 |
| `@Embedded` | 嵌入对象 | 把一个对象的字段展平到表中 |
| `@Ignore` | 忽略字段 | 该字段不存入数据库 |
| `@ForeignKey` | 外键约束 | 定义表间关系 |
| `@Index` | 索引 | 加速查询 |
| `@Dao` | 数据访问接口 | 定义查询方法 |
| `@Query` | 查询 | 支持 SQL 查询语句，支持 Flow/LiveData 返回 |
| `@Insert` | 插入 | 支持 `OnConflictStrategy` 冲突策略 |
| `@Update` / `@Delete` | 更新/删除 | 支持按实体操作 |
| `@Relation` | 关系查询 | 在 POJO 中定义一对多、多对多关系 |
| `@Database` | 数据库定义 | 列出所有 Entity、版本号 |
| `@TypeConverter` | 类型转换 | 自定义类型 ↔ 数据库类型的转换 |
| `Room.databaseBuilder()` | 构建数据库 | 创建单例数据库实例 |
| `Migration` | 数据库迁移 | 版本升级时的数据迁移逻辑 |

#### Gradle 依赖

```groovy
dependencies {
    implementation "androidx.room:room-runtime:2.6.1"
    annotationProcessor "androidx.room:room-compiler:2.6.1"
    // 可选：LiveData 支持
    implementation "androidx.room:room-livedata:2.6.1"
    // 可选：RxJava 支持
    implementation "androidx.room:room-rxjava3:2.6.1"
    // 可选：Paging 支持
    implementation "androidx.room:room-paging:2.6.1"
}
```

#### 代码示例

**基础用法（完整流程）：**

```java
// 1. 定义实体（对应一张表）
@Entity(tableName = "users")
public class User {
    @PrimaryKey(autoGenerate = true)
    private int uid;

    @ColumnInfo(name = "first_name")
    private String firstName;

    @ColumnInfo(name = "last_name")
    private String lastName;

    private int age;

    @Ignore  // 这个字段不存入数据库
    private transient String tempFlag;

    // Room 要求必须有构造函数（可以是全部参数或部分参数）
    // 全参构造（Room 用这个来从 Cursor 创建对象）
    public User(int uid, String firstName, String lastName, int age) {
        this.uid = uid;
        this.firstName = firstName;
        this.lastName = lastName;
        this.age = age;
    }

    // 无参构造（可选，方便手动创建）
    public User() {}

    // Getter 和 Setter（Room 通过 getter/setter 访问字段）
    public int getUid() { return uid; }
    public void setUid(int uid) { this.uid = uid; }
    public String getFirstName() { return firstName; }
    public void setFirstName(String firstName) { this.firstName = firstName; }
    public String getLastName() { return lastName; }
    public void setLastName(String lastName) { this.lastName = lastName; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}

// 2. 定义 DAO（数据访问对象）
@Dao
public interface UserDao {

    // 查询所有用户，返回 LiveData（数据变化时自动通知）
    @Query("SELECT * FROM users ORDER BY last_name ASC")
    LiveData<List<User>> getAllUsers();

    // 按条件查询
    @Query("SELECT * FROM users WHERE age > :minAge")
    List<User> getUsersOlderThan(int minAge);

    // 按名字模糊搜索
    @Query("SELECT * FROM users WHERE first_name LIKE '%' || :keyword || '%'")
    LiveData<List<User>> searchUsers(String keyword);

    // 插入单条，冲突时替换
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    long insert(User user);

    // 批量插入
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    List<Long> insertAll(List<User> users);

    // 更新
    @Update
    int update(User user);

    // 删除
    @Delete
    int delete(User user);

    // 复杂查询：联表
    @Query("SELECT users.*, orders.order_count FROM users "
         + "LEFT JOIN (SELECT user_id, COUNT(*) as order_count FROM orders GROUP BY user_id) orders "
         + "ON users.uid = orders.user_id "
         + "WHERE users.age > :minAge")
    LiveData<List<UserWithOrderCount>> getUsersWithOrderCount(int minAge);
}

// 3. 定义数据库
@Database(entities = {User.class}, version = 1, exportSchema = false)
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}

// 4. 创建数据库单例
public class DatabaseHelper {

    private static volatile AppDatabase INSTANCE;

    public static AppDatabase getInstance(Context context) {
        if (INSTANCE == null) {
            synchronized (DatabaseHelper.class) {
                if (INSTANCE == null) {
                    INSTANCE = Room.databaseBuilder(
                        context.getApplicationContext(),
                        AppDatabase.class,
                        "app_database"
                    )
                    // .addMigrations(MIGRATION_1_2)  // 数据库升级迁移
                    // .fallbackToDestructiveMigration()  // 谨慎使用！会清空数据
                    .build();
                }
            }
        }
        return INSTANCE;
    }
}

// 5. 在 ViewModel 中使用
public class UserListViewModel extends ViewModel {

    private final LiveData<List<User>> userList;
    private final UserDao userDao;
    private final ExecutorService executor = Executors.newSingleThreadExecutor();

    public UserListViewModel(Application application) {
        AppDatabase db = DatabaseHelper.getInstance(application);
        userDao = db.userDao();
        userList = userDao.getAllUsers(); // 返回 LiveData，自动监听变化
    }

    public LiveData<List<User>> getUserList() { return userList; }

    public void addUser(User user) {
        executor.execute(() -> userDao.insert(user)); // 数据库操作不能在主线程
    }
}
```

**数据库迁移（版本升级）：**

```java
// Migration：从版本 1 升级到版本 2，新增 email 字段
static final Migration MIGRATION_1_2 = new Migration(1, 2) {
    @Override
    public void migrate(@NonNull SupportSQLiteDatabase database) {
        database.execSQL("ALTER TABLE users ADD COLUMN email TEXT DEFAULT ''");
    }
};

// 版本 2 升级到版本 3，新增索引
static final Migration MIGRATION_2_3 = new Migration(2, 3) {
    @Override
    public void migrate(@NonNull SupportSQLiteDatabase database) {
        database.execSQL("CREATE INDEX index_users_email ON users(email)");
    }
};

// 构建数据库时添加迁移
Room.databaseBuilder(context, AppDatabase.class, "app_database")
    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
    .build();
```

**TypeConverter（自定义类型转换）：**

```java
// 场景：Date 类型不能直接存入数据库，需要转换
public class Converters {

    @TypeConverter
    public static Date fromTimestamp(Long value) {
        return value == null ? null : new Date(value);
    }

    @TypeConverter
    public static Long dateToTimestamp(Date date) {
        return date == null ? null : date.getTime();
    }

    // List<String> ↔ JSON 字符串
    @TypeConverter
    public static String fromStringList(List<String> list) {
        if (list == null) return null;
        return new Gson().toJson(list);
    }

    @TypeConverter
    public static List<String> toStringList(String json) {
        if (json == null) return null;
        return new Gson().fromJson(json, new TypeToken<List<String>>() {}.getType());
    }
}

// 在 @Database 中注册 TypeConverter
@Database(entities = {User.class}, version = 1)
@TypeConverters({Converters.class})
public abstract class AppDatabase extends RoomDatabase {
    public abstract UserDao userDao();
}
```

**关系查询（@Relation + @Embedded）：**

```java
// 一对多：一个用户有多个订单
public class UserWithOrders {
    @Embedded
    public User user;

    @Relation(
        parentColumn = "uid",
        entityColumn = "user_id"
    )
    public List<Order> orders;
}

// 在 DAO 中查询
@Dao
public interface UserDao {
    @Transaction  // 关系查询必须加 @Transaction，保证原子性
    @Query("SELECT * FROM users WHERE uid = :userId")
    LiveData<UserWithOrders> getUserWithOrders(int userId);
}
```

#### 最佳实践

- 数据库操作**绝对不要在主线程**执行，Room 默认会抛 `IllegalStateException`。用 `ExecutorService`、`RxJava` 或返回 `LiveData`（Room 自动在后台线程执行）
- 升级数据库**必须用 Migration**，千万别用 `fallbackToDestructiveMigration()`（会清空所有数据）。在开发阶段可以暂时用，但上线前一定要删掉
- 数据库实例用**单例模式**，整个 App 只创建一个 RoomDatabase 实例
- 大量数据用 `Paging` + Room 配合（DAO 返回 `DataSource.Factory`），别一次性全查出来
- `@Database` 的 `exportSchema` 建议设为 `true`，把 schema JSON 文件保存到版本控制中，方便追踪数据库结构变化

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 主线程崩溃 | 数据库操作在主线程执行 | 用 Executor 或返回 LiveData |
| 升级后崩溃 | 没提供 Migration | 添加对应的 Migration |
| 实体类编译报错 | 缺少 Getter/Setter 或构造函数 | Room 需要空构造或全参构造 + getter/setter |
| @Relation 查询结果为空 | 没加 @Transaction | 关系查询必须加 @Transaction |
| 查询结果字段为 null | 列名和字段名不匹配 | 用 @ColumnInfo 映射列名 |

---

### 4. DataStore

#### 一句话定位

SharedPreferences 的替代品，全程异步不会卡主线程，支持事务性写入。

#### 解决了什么问题

SharedPreferences（SP）有几个老毛病：
- `apply()` 看似异步，实际上是在主线程做 XML 序列化，数据量大时会 ANR
- `commit()` 直接同步写磁盘，更危险
- 没有类型安全，什么类型都能塞进去，取出来类型不对就崩
- `apply()` 的原子性有问题——进程意外死亡时可能丢失最后一次写入

DataStore 用协程（Kotlin）或 Executor（Java）重新做了一遍，彻底解决了这些问题。

#### 适用场景

- 存储用户设置（主题模式、语言、通知开关等简单键值对）
- 存储用户 Token、登录状态等少量配置信息
- 替代 SharedPreferences 的所有场景
- 需要类型安全的配置存储

#### 核心原理

打个比方：SP 像一个小本子，每次读写都得等上一个人写完（apply 看似异步其实有坑）。DataStore 像一个自动存取柜——你把东西放进去（`updateDataAsync`），它保证原子性操作（要么全成功要么全失败），而且全程异步，绝不让你干等。

技术层面：

- **Preferences DataStore**：键值对存储，类似 SP 的用法，但用 Flow/LiveData 暴露数据变化
- **Proto DataStore**：存自定义类型对象，需要定义 Protocol Buffers schema（目前 Java 支持有限）
- Java 中使用 DataStore 主要通过 `PreferencesDataStoreFactory` 和 `RxDataStore`（RxJava3 版本）
- `updateDataAsync()` 内部是原子性的读-改-写事务，并发安全
- 数据存在文件里（XML 格式），通过异步 IO 读写
- 同一进程中对同一个文件只应创建一个 DataStore 实例

#### Gradle 依赖

```groovy
dependencies {
    // Preferences DataStore（Java 推荐用 RxJava 版本）
    implementation "androidx.datastore:datastore-preferences:1.1.1"
    implementation "androidx.datastore:datastore-preferences-rxjava3:1.1.1"
    // 如果需要 RxJava
    implementation "io.reactivex.rxjava3:rxjava:3.1.9"
    implementation "io.reactivex.rxjava3:rxandroid:3.0.2"
}
```

#### 代码示例

**Java + RxJava3 使用 Preferences DataStore：**

```java
// 1. 创建 DataStore 单例
public class SettingsDataStore {

    private static volatile SettingsDataStore INSTANCE;
    private final RxPreferenceDataStore<Preferences> dataStore;

    // 定义键
    public static final Preferences.Key<Boolean> DARK_MODE_KEY =
            Preferences.Key.booleanPreferencesKey("dark_mode");
    public static final Preferences.Key<String> USER_TOKEN_KEY =
            Preferences.Key.stringPreferencesKey("user_token");
    public static final Preferences.Key<Integer> PAGE_SIZE_KEY =
            Preferences.Key.intPreferencesKey("page_size");

    private SettingsDataStore(Context context) {
        dataStore = new RxPreferenceDataStoreBuilder(
            context.getApplicationContext(),
            /* name = */ "settings"
        ).build();
    }

    public static SettingsDataStore getInstance(Context context) {
        if (INSTANCE == null) {
            synchronized (SettingsDataStore.class) {
                if (INSTANCE == null) {
                    INSTANCE = new SettingsDataStore(context);
                }
            }
        }
        return INSTANCE;
    }

    // 读取（返回 Flowable，数据变化时自动通知）
    public Flowable<Boolean> isDarkMode() {
        return dataStore.data()
            .map(prefs -> prefs.get(DARK_MODE_KEY) != null
                ? prefs.get(DARK_MODE_KEY) : false)
            .subscribeOn(Schedulers.io());
    }

    public Flowable<String> getUserToken() {
        return dataStore.data()
            .map(prefs -> prefs.get(USER_TOKEN_KEY) != null
                ? prefs.get(USER_TOKEN_KEY) : "")
            .subscribeOn(Schedulers.io());
    }

    // 写入（原子性操作）
    public Completable setDarkMode(boolean enabled) {
        return dataStore.updateDataAsync(prefsIn -> {
            MutablePreferences mutablePrefs = prefsIn.toMutablePreferences();
            mutablePrefs.set(DARK_MODE_KEY, enabled);
            return Single.just(mutablePrefs);
        }).subscribeOn(Schedulers.io());
    }

    public Completable setUserToken(String token) {
        return dataStore.updateDataAsync(prefsIn -> {
            MutablePreferences mutablePrefs = prefsIn.toMutablePreferences();
            mutablePrefs.set(USER_TOKEN_KEY, token);
            return Single.just(mutablePrefs);
        }).subscribeOn(Schedulers.io());
    }

    // 清空所有数据
    public Completable clearAll() {
        return dataStore.updateDataAsync(prefsIn ->
            Single.just(prefsIn.toMutablePreferences().clear())
        ).subscribeOn(Schedulers.io());
    }
}

// 2. 在 Activity/ViewModel 中使用
public class SettingsViewModel extends ViewModel {

    private final SettingsDataStore dataStore;

    public SettingsViewModel(Application application) {
        dataStore = SettingsDataStore.getInstance(application);
    }

    public Flowable<Boolean> isDarkMode() {
        return dataStore.isDarkMode();
    }

    public void setDarkMode(boolean enabled) {
        dataStore.setDarkMode(enabled)
            .subscribe(
                () -> { /* 成功 */ },
                error -> { /* 错误处理 */ }
            );
    }
}

// Activity 中观察
public class SettingsActivity extends AppCompatActivity {

    private SettingsViewModel viewModel;
    private final CompositeDisposable disposables = new CompositeDisposable();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        viewModel = new ViewModelProvider(this).get(SettingsViewModel.class);

        disposables.add(viewModel.isDarkMode()
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(darkMode -> {
                darkModeSwitch.setChecked(darkMode);
            }));

        darkModeSwitch.setOnCheckedChangeListener((buttonView, isChecked) -> {
            viewModel.setDarkMode(isChecked);
        });
    }

    @Override
    protected void onDestroy() {
        disposables.clear(); // 防止泄漏
        super.onDestroy();
    }
}
```

**从 SharedPreferences 迁移到 DataStore：**

```java
// DataStore 支持自动迁移 SP 数据
RxPreferenceDataStoreBuilder(context, "settings")
    .setMigration(new SharedPreferencesMigration<>(context, "old_shared_prefs_name"))
    .build();
```

#### 最佳实践

- DataStore 实例用**单例模式**，同一进程中对同一个文件只创建一个实例，否则会出并发问题
- Java 项目推荐用 `datastore-preferences-rxjava3` 版本，API 更友好
- 简单配置用 Preferences DataStore，复杂对象可以考虑 Proto DataStore 或 Room
- SP → DataStore 迁移有专门的 `SharedPreferencesMigration`，不要手动搬数据
- 写入操作返回 `Completable`，记得在 `onDestroy` 中清理 RxJava 订阅

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 并发修改异常 | 创建了多个 DataStore 实例 | 用单例模式 |
| 数据丢失 | 进程在写入时被杀 | DataStore 保证原子性，比 SP 安全 |
| RxJava 泄漏 | 没有在 onDestroy 取消订阅 | 用 CompositeDisposable 管理 |

---

### 5. Navigation

#### 一句话定位

管理页面跳转的"总调度"，用图结构替代碎片化的事务管理。

#### 解决了什么问题

以前用 `FragmentManager.beginTransaction()` 管理页面跳转，代码又散又乱：
- 传参靠 Bundle，key 写错了运行时才崩
- 深链接（从浏览器/通知跳转到特定页面）实现复杂
- 返回栈管理混乱，动画不统一
- 过渡动画需要在每个事务里手动设置

Navigation 把所有页面和跳转关系画成一张"地图"（NavGraph），跳转、传参、返回栈、深链接、动画全部统一管理。

#### 适用场景

- App 内多页面导航（Tab 切换、列表→详情、登录→主页等）
- 需要深链接支持（从外部跳转到 App 内特定页面）
- 复杂的导航流程（条件跳转、嵌套导航图）
- 需要统一的动画和转场效果
- 底部导航栏 + Fragment 切换

#### 核心原理

打个比方：Navigation 就像商场导航图。你站在商场入口（NavHost），想去三楼餐厅（Destination），导航图告诉你怎么走（NavController.navigate()），还能自动规划路线（深链接直达）。你走过的路它都记着（返回栈），随时可以原路返回。

技术层面：

- **NavGraph**：图结构，定义所有目的地（Destination）和它们之间的连线（Action）。可以在 XML 中定义，也可以用 Navigation Component 的 DSL 动态创建
- **NavController**：中央调度器，执行 navigate、popBackStack、navigateUp 等操作。内部维护一个返回栈（BackStack）
- **NavHostFragment**：UI 容器 Fragment，根据当前目的地显示对应的 Fragment
- **Safe Args**：Gradle 插件，为每个目的地生成类型安全的参数类，传参不会写错类型和 key
- **Deep Link**：支持从 URL、通知、Intent 直接跳转到指定目的地
- Navigation 跟 Fragment 生命周期绑定，自动处理返回键

**导航流程：**

```
用户点击 → NavController.navigate(actionId, bundle)
  → 查找 NavGraph 中的 Action
  → 确定目标 Destination
  → 执行 FragmentTransaction（replace）
  → 将当前 Destination 压入返回栈
  → 显示目标 Fragment
```

#### 关键 API

| API | 用途 | 说明 |
|-----|------|------|
| `NavHostFragment` | 导航容器 | 放在布局中，自动显示当前目的地 |
| `NavController` | 导航控制器 | 执行跳转、返回等操作 |
| `Navigation.findNavController(view)` | 获取 NavController | 从 View 获取关联的 NavController |
| `navController.navigate(actionId)` | 按Action跳转 | 最常用的跳转方式 |
| `navController.navigate(actionId, bundle)` | 带参数跳转 | 通过 Bundle 传参 |
| `navController.navigateUp()` | 向上一级 | 类似返回键 |
| `navController.popBackStack()` | 弹出返回栈 | 返回上一页 |
| `navController.popBackStack(id, inclusive)` | 弹到指定页面 | inclusive=true 连目标也弹出 |
| `navController.getBackStackEntry()` | 获取返回栈条目 | 可以获取关联的 ViewModel |
| `NavDirections` | Safe Args 生成的类 | 类型安全的跳转和参数 |
| `navController.navigate(deepLink)` | 深链接跳转 | 从 URI/Intent 跳转 |

#### Gradle 依赖

```groovy
dependencies {
    implementation "androidx.navigation:navigation-fragment:2.8.5"
    implementation "androidx.navigation:navigation-ui:2.8.5"
    // Safe Args 插件（在 build.gradle 顶层）
}

// build.gradle（项目级）
plugins {
    id 'androidx.navigation.safeargs' version '2.8.5' apply false
}

// build.gradle（模块级）
plugins {
    id 'androidx.navigation.safeargs'
}
```

#### 代码示例

**1. 定义导航图（res/navigation/main_nav.xml）：**

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main_nav"
    app:startDestination="@id/homeFragment">

    <!-- 首页 -->
    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.app.HomeFragment"
        tools:layout="@layout/fragment_home">
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment"
            app:enterAnim="@anim/slide_in_right"
            app:exitAnim="@anim/slide_out_left"
            app:popEnterAnim="@anim/slide_in_left"
            app:popExitAnim="@anim/slide_out_right" />
    </fragment>

    <!-- 详情页（带参数） -->
    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.app.DetailFragment"
        tools:layout="@layout/fragment_detail">
        <argument
            android:name="itemId"
            app:argType="string" />
        <argument
            android:name="from"
            android:defaultValue="home"
            app:argType="string" />
        <deepLink
            android:id="@+id/deepLink"
            app:uri="https://www.example.com/detail/{itemId}" />
    </fragment>

    <!-- 设置页 -->
    <fragment
        android:id="@+id/settingsFragment"
        android:name="com.example.app.SettingsFragment"
        tools:layout="@layout/fragment_settings" />

</navigation>
```

**2. 在 Activity 布局中放置 NavHostFragment：**

```xml
<!-- activity_main.xml -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <fragment
        android:id="@+id/nav_host_fragment"
        android:name="androidx.navigation.fragment.NavHostFragment"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        app:defaultNavHost="true"
        app:navGraph="@navigation/main_nav" />

    <!-- 底部导航栏 -->
    <com.google.android.material.bottomnavigation.BottomNavigationView
        android:id="@+id/bottom_nav"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        app:menu="@menu/bottom_nav_menu" />

</LinearLayout>
```

**3. 在 Activity 中配置底部导航：**

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        BottomNavigationView bottomNav = findViewById(R.id.bottom_nav);
        NavController navController = Navigation.findNavController(this, R.id.nav_host_fragment);

        // 底部导航栏自动绑定 Navigation
        NavigationUI.setupWithNavController(bottomNav, navController);

        // 处理 ActionBar 的返回箭头
        AppBarConfiguration appBarConfig = new AppBarConfiguration.Builder(
            R.id.homeFragment, R.id.settingsFragment  // 这些是顶层页面，不显示返回箭头
        ).build();
        NavigationUI.setupActionBarWithNavController(this, navController, appBarConfig);
    }

    // 处理返回键
    @Override
    public boolean onSupportNavigateUp() {
        NavController navController = Navigation.findNavController(this, R.id.nav_host_fragment);
        return navController.navigateUp() || super.onSupportNavigateUp();
    }
}
```

**4. 在 Fragment 中跳转（使用 Safe Args）：**

```java
public class HomeFragment extends Fragment {

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        NavController navController = Navigation.findNavController(view);

        // 跳转到详情页，Safe Args 自动生成 HomeFragmentDirections
        view.findViewById(R.id.btn_go_detail).setOnClickListener(v -> {
            HomeFragmentDirections.ActionHomeToDetail action =
                HomeFragmentDirections.actionHomeToDetail("item_123");
            action.setFrom("home"); // 设置可选参数
            navController.navigate(action);
        });
    }
}

// 在目标 Fragment 中接收参数
public class DetailFragment extends Fragment {

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        // Safe Args 自动生成参数类
        DetailFragmentArgs args = DetailFragmentArgs.fromBundle(getArguments());
        String itemId = args.getItemId();
        String from = args.getFrom();
    }
}
```

**5. 深链接（从外部跳转）：**

```java
// 在 Activity 的 onCreate 中处理深链接 Intent
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    NavController navController = Navigation.findNavController(this, R.id.nav_host_fragment);

    // 自动处理 NavGraph 中定义的 deepLink
    navController.handleDeepLink(getIntent());
}
```

#### 最佳实践

- 导航图用 XML 定义（可视化编辑），复杂场景可以用代码动态创建
- 传参**一定要用 Safe Args**，别手写 Bundle key，写错了运行时才崩
- 避免在 ViewModel 里持有 NavController，会内存泄漏。NavController 应该只在 UI 层使用
- 嵌套导航图（Navigation）用于模块化：比如底部导航的每个 Tab 有自己的子导航图
- `app:defaultNavHost="true"` 让 NavHostFragment 拦截返回键，否则返回键会直接关闭 Activity

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 返回键直接关了 Activity | 没设 defaultNavHost 或没重写 onSupportNavigateUp | 设置 defaultNavHost="true" 并重写 onSupportNavigateUp |
| 参数传错了 | 用了 Bundle 而不是 Safe Args | 用 Safe Args 插件 |
| 底部导航切换时 Fragment 重建 | 没用正确的导航图结构 | 用嵌套导航图 + bottom navigation 模式 |
| NavController 内存泄漏 | 在 ViewModel 中持有了 NavController | NavController 只在 UI 层使用 |

---

### 6. Paging 3

#### 一句话定位

分页加载的"全家桶"，滚动到底自动加载下一页，内置缓存和错误处理。

#### 解决了什么问题

加载大量数据（朋友圈、商品列表、搜索结果），不可能一次性全查出来。你得自己管理页码、处理加载状态（加载中/加载失败/没有更多）、防止重复请求、做内存缓存……Paging 3 把这些全包了，你只需要告诉它"数据从哪来"。

#### 适用场景

- 列表数据量大，需要分页加载（网络分页、数据库分页）
- 需要自动预加载（用户还没滚到底就开始加载下一页）
- 需要网络 + 本地缓存的分层数据源（先显示缓存，再更新网络数据）
- 需要统一的加载状态管理（Loading、Error、Empty、Append、Prepend）

#### 核心原理

打个比方：Paging 就像自动续杯的咖啡机。杯子（UI）快喝完了，咖啡机（PagingSource）自动再倒一杯。你不用管什么时候该续、续多少、咖啡机忙不忙，它自己全处理好了。

技术层面——三层架构：

```
┌──────────────────────────────────────────────────────┐
│                    UI 层                              │
│  PagingDataAdapter (RecyclerView)                     │
│  自动绑定数据、显示加载状态、处理空列表                 │
├──────────────────────────────────────────────────────┤
│                  ViewModel 层                         │
│  Pager + PagingConfig                                 │
│  根据配置生成 PagingData 流，协调数据加载              │
├──────────────────────────────────────────────────────┤
│                 Repository 层                         │
│  PagingSource（数据源）                               │
│  RemoteMediator（网络+本地缓存协调）                   │
│  定义怎么加载某一页数据                                │
└──────────────────────────────────────────────────────┘
```

- **PagingSource<Key, Value>**：数据源，`Key` 是页码类型（Integer、String），`Value` 是数据类型。定义 `load()` 方法，返回某一页的数据
- **RemoteMediator**：处理"网络 + 本地缓存"的分层数据源。先显示 Room 缓存，后台请求网络，网络成功后更新数据库，数据库变化自动触发 UI 更新
- **Pager**：根据 `PagingConfig`（页大小、预取距离等）生成 `PagingData` 流
- **PagingDataAdapter**：RecyclerView 适配器，内置 `AsyncPagingDataDiffer`（内部用 DiffUtil），自动计算差异并更新
- 内置内存缓存，已加载的页面不会重复请求
- 自动处理加载状态：`LoadState`（Loading、NotLoading、Error）分为 refresh（首次加载）、append（加载更多）、prepend（向上加载）

#### Gradle 依赖

```groovy
dependencies {
    implementation "androidx.paging:paging-runtime:3.3.6"
    // 可选：RxJava 支持
    implementation "androidx.paging:paging-rxjava3:3.3.6"
}
```

#### 代码示例

**基础用法（网络分页）：**

```java
// 1. 定义数据源
public class ArticlePagingSource extends PagingSource<Integer, Article> {

    private final ArticleApi api;

    public ArticlePagingSource(ArticleApi api) {
        this.api = api;
    }

    @Nullable
    @Override
    public Key getRefreshKey(@NonNull PagingState<Integer, Article> state) {
        // 下拉刷新时从哪一页开始
        Integer anchorPosition = state.getAnchorPosition();
        if (anchorPosition == null) return null;

        LoadResult.Page<Integer, Article> anchorPage = state.closestPageToPosition(anchorPosition);
        if (anchorPage == null) return null;

        return anchorPage.getPrevKey() != null
            ? anchorPage.getPrevKey() + 1
            : anchorPage.getNextKey() - 1;
    }

    @NonNull
    @Override
    public Single<LoadResult<Integer, Article>> loadSingle(@NonNull LoadParams<Integer> params) {
        int page = params.getKey() != null ? params.getKey() : 1;
        int loadSize = params.getLoadSize();

        return api.getArticles(page, loadSize)
            .map(response -> {
                List<Article> articles = response.getArticles();
                int nextKey = articles.isEmpty() ? null : page + 1;
                int prevKey = page == 1 ? null : page - 1;
                return new LoadResult.Page<>(articles, prevKey, nextKey);
            })
            .onErrorReturn(LoadResult.Error::new);
    }
}

// 2. 定义 Adapter
public class ArticleAdapter extends PagingDataAdapter<Article, ArticleViewHolder> {

    public ArticleAdapter() {
        super(new DiffUtil.ItemCallback<Article>() {
            @Override
            public boolean areItemsTheSame(@NonNull Article oldItem, @NonNull Article newItem) {
                return oldItem.getId().equals(newItem.getId());
            }

            @Override
            public boolean areContentsTheSame(@NonNull Article oldItem, @NonNull Article newItem) {
                return oldItem.equals(newItem);
            }
        });
    }

    @NonNull
    @Override
    public ArticleViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        ItemArticleBinding binding = ItemArticleBinding.inflate(
            LayoutInflater.from(parent.getContext()), parent, false);
        return new ArticleViewHolder(binding);
    }

    @Override
    public void onBindViewHolder(@NonNull ArticleViewHolder holder, int position) {
        Article article = getItem(position);
        if (article != null) {
            holder.bind(article);
        }
    }
}

// 3. ViewModel 中构建
public class ArticleListViewModel extends ViewModel {

    private final PagingConfig config = new PagingConfig(
        20,     // pageSize：每页多少条
        5,      // prefetchDistance：距离底部多少条时预加载
        false,  // enablePlaceholders：是否显示占位符
        20      // initialLoadSize：首次加载多少条
    );

    private final RxPagingSource<Integer, Article> pagingSource;
    private final Flowable<PagingData<Article>> pagingDataFlow;

    public ArticleListViewModel(ArticleApi api) {
        ArticlePagingSource source = new ArticlePagingSource(api);
        Pager<Integer, Article> pager = new Pager<>(config, () -> source);
        pagingDataFlow = Pager.createFlowable(config, () -> source);
    }

    public Flowable<PagingData<Article>> getPagingData() {
        return pagingDataFlow;
    }
}

// 4. Activity 中使用
public class ArticleListActivity extends AppCompatActivity {

    private ArticleAdapter adapter = new ArticleAdapter();
    private final CompositeDisposable disposables = new CompositeDisposable();

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_article_list);

        RecyclerView recyclerView = findViewById(R.id.recycler_view);
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        recyclerView.setAdapter(adapter);

        // 添加加载状态监听
        adapter.addLoadStateListener(loadStates -> {
            LoadState refresh = loadStates.getRefresh();
            LoadState append = loadStates.getAppend();

            if (refresh instanceof LoadState.Loading) {
                // 首次加载中
                progressBar.setVisibility(View.VISIBLE);
            } else if (refresh instanceof LoadState.Error) {
                // 首次加载失败
                progressBar.setVisibility(View.GONE);
                errorView.setVisibility(View.VISIBLE);
                String errorMsg = ((LoadState.Error) refresh).getError().getMessage();
                errorText.setText("加载失败: " + errorMsg);
            } else {
                // 加载成功
                progressBar.setVisibility(View.GONE);
                errorView.setVisibility(View.GONE);
            }

            if (append instanceof LoadState.Loading) {
                // 加载更多中（显示底部 loading）
                footerLoading.setVisibility(View.VISIBLE);
            } else {
                footerLoading.setVisibility(View.GONE);
            }
        });

        // 重试按钮
        retryButton.setOnClickListener(v -> adapter.retry());

        // 观察 PagingData
        ArticleListViewModel viewModel = new ViewModelProvider(this).get(ArticleListViewModel.class);
        disposables.add(
            viewModel.getPagingData()
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(pagingData -> adapter.submitData(getLifecycle(), pagingData))
        );
    }

    @Override
    protected void onDestroy() {
        disposables.clear();
        super.onDestroy();
    }
}
```

**带 LoadState Footer 的 Adapter：**

```java
// 在列表底部显示加载状态
public class ArticleAdapterWithFooter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    private static final int TYPE_ITEM = 0;
    private static final int TYPE_FOOTER = 1;

    private final ArticleAdapter adapter;
    private final LoadStateAdapter<RecyclerView.ViewHolder> footerAdapter;

    // 使用 ConcatAdapter 合并数据 Adapter 和 Footer Adapter
    public ConcatAdapter getConcatAdapter() {
        return new ConcatAdapter(adapter, footerAdapter);
    }
}

// Footer Adapter
public class ArticleLoadStateAdapter extends LoadStateAdapter<RecyclerView.ViewHolder> {

    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, @NonNull LoadState loadState) {
        ItemLoadStateFooterBinding binding = ItemLoadStateFooterBinding.inflate(
            LayoutInflater.from(parent.getContext()), parent, false);
        return new FooterViewHolder(binding);
    }

    @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, @NonNull LoadState loadState) {
        FooterViewHolder vh = (FooterViewHolder) holder;
        if (loadState instanceof LoadState.Loading) {
            vh.binding.progressBar.setVisibility(View.VISIBLE);
            vh.binding.errorText.setVisibility(View.GONE);
            vh.binding.retryButton.setVisibility(View.GONE);
        } else if (loadState instanceof LoadState.Error) {
            vh.binding.progressBar.setVisibility(View.GONE);
            vh.binding.errorText.setVisibility(View.VISIBLE);
            vh.binding.retryButton.setVisibility(View.VISIBLE);
        } else {
            vh.binding.progressBar.setVisibility(View.GONE);
            vh.binding.errorText.setVisibility(View.GONE);
            vh.binding.retryButton.setVisibility(View.GONE);
        }
    }
}
```

#### 最佳实践

- `pageSize` 建议设为 `prefetchDistance` 的 2~3 倍，确保用户滚到底之前数据已经加载好
- 网络分页用 `PagingSource`，网络 + 本地缓存用 `RemoteMediator`
- Java 项目推荐用 `paging-rxjava3` 版本，API 更友好
- `PagingDataAdapter.submitData()` 需要传入 `Lifecycle`，确保在页面可见时才更新数据
- `adapter.retry()` 可以方便地重试失败的加载

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 数据闪烁 | submitData 被多次调用 | 确保 PagingData 只提交一次 |
| 加载状态不更新 | 没有通过 addLoadStateListener 监听 | 注册 LoadState 监听器 |
| RxJava 版本不兼容 | 混用了 RxJava2 和 RxJava3 | 统一用 RxJava3 |

---

### 7. Hilt（Dagger Hilt）

#### 一句话定位

Android 版的依赖注入工具，自动帮你把需要的对象"注入"到正确的地方。

#### 解决了什么问题

一个 Activity 可能需要 Repository、ViewModel、网络客户端、数据库……以前你得手动 new 出来一层层传递，或者写单例，代码耦合严重还不好测试。Hilt 让你声明"我需要什么"，它自动帮你创建和注入。

#### 适用场景

- 需要在多个地方共享同一个实例（如 Retrofit、OkHttpClient、Database）
- 需要解耦以便单元测试（可以注入 Mock 对象）
- ViewModel、Repository、Service 等需要依赖外部资源
- 大型项目，手动管理依赖太痛苦

#### 核心原理

打个比方：Hilt 就像外卖平台。你（Activity）只需要在门口贴个单子说"我要一杯咖啡"（@Inject），外卖平台（Hilt）会自动找到能做咖啡的店（@Module），做好送到你手上。你不用管咖啡怎么做、用哪家店、送餐路线怎么走。

技术层面：

- 编译时通过 KSP 生成 Dagger 组件代码（不是运行时反射，所以性能好）
- `@HiltAndroidApp` 在 Application 上触发，生成应用级依赖容器（根组件）
- `@AndroidEntryPoint` 给每个 Android 类生成子容器，从父组件获取依赖
- **组件层级**（从上到下，子组件可以访问父组件的依赖）：

```
SingletonComponent（Application 级，生命周期 = App 进程）
  ├── ActivityComponent（Activity 级，生命周期 = Activity）
  │     ├── FragmentComponent（Fragment 级，生命周期 = Fragment 视图）
  │     └── ViewComponent（View 级，生命周期 = View）
  ├── ServiceComponent（Service 级，生命周期 = Service）
  └── ViewModelComponent（ViewModel 级，生命周期 = ViewModel）
```

- `@Inject constructor` 告诉 Hilt "这个类可以自动创建"
- `@Module + @Provides` 告诉 Hilt "这个类的创建方式比较复杂（第三方库、接口等），按我说的来"
- `@Binds` 告诉 Hilt "这个接口的实现类是这个"（比 @Provides 更高效）
- `@Qualifier` 用于区分同一类型的不同实例（如两个不同的 OkHttpClient）

#### Gradle 依赖

```groovy
dependencies {
    implementation 'com.google.dagger:hilt-android:2.51.1'
    annotationProcessor 'com.google.dagger:hilt-compiler:2.51.1'
}

// build.gradle（项目级）
plugins {
    id 'com.google.dagger.hilt.android' version '2.51.1' apply false
}

// build.gradle（模块级）
plugins {
    id 'com.google.dagger.hilt.android'
}
```

#### 代码示例

**基础用法：**

```java
// 1. Application 类上加 @HiltAndroidApp
@HiltAndroidApp
public class MyApplication extends Application {
}

// 2. 需要注入的类（构造函数注入）
public class AnalyticsService {

    private final Retrofit retrofit;

    @Inject  // 告诉 Hilt：这个类可以通过构造函数创建
    public AnalyticsService(Retrofit retrofit) {
        this.retrofit = retrofit;
    }

    public void trackEvent(String event) {
        // ...
    }
}

// 3. 提供第三方库实例的 Module
@Module
@InstallIn(SingletonComponent.class)  // 绑定到 Application 级别
public class NetworkModule {

    @Provides
    @Singleton  // 全局单例
    public OkHttpClient provideOkHttpClient() {
        return new OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .addInterceptor(new HttpLoggingInterceptor().setLevel(HttpLoggingInterceptor.Level.BODY))
            .build();
    }

    @Provides
    @Singleton
    public Retrofit provideRetrofit(OkHttpClient okHttpClient) {
        // 参数 okHttpClient 会自动从上面的 provideOkHttpClient() 获取
        return new Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .client(okHttpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build();
    }

    @Provides
    @Singleton
    public ArticleApi provideArticleApi(Retrofit retrofit) {
        return retrofit.create(ArticleApi.class);
    }
}

// 4. 在 Activity 中使用（字段注入）
@AndroidEntryPoint
public class MainActivity extends AppCompatActivity {

    @Inject  // 字段注入，Hilt 会自动赋值
    AnalyticsService analyticsService;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 此时 analyticsService 已经被注入了
        analyticsService.trackEvent("page_view");
    }
}

// 5. 在 Fragment 中使用
@AndroidEntryPoint
public class UserFragment extends Fragment {

    @Inject
    AnalyticsService analyticsService;

    // ...
}

// 6. 在 ViewModel 中使用（必须用 @HiltViewModel）
@HiltViewModel
public class UserViewModel extends ViewModel {

    private final UserRepository repository;
    private final AnalyticsService analyticsService;

    @Inject  // ViewModel 的构造函数注入
    public UserViewModel(UserRepository repository, AnalyticsService analyticsService) {
        this.repository = repository;
        this.analyticsService = analyticsService;
    }
}
```

**接口绑定（@Binds）：**

```java
// 定义接口
public interface UserRepository {
    LiveData<User> getUser(String userId);
}

// 实现类
public class UserRepositoryImpl implements UserRepository {

    private final ArticleApi api;
    private final AppDatabase db;

    @Inject
    public UserRepositoryImpl(ArticleApi api, AppDatabase db) {
        this.api = api;
        this.db = db;
    }

    @Override
    public LiveData<User> getUser(String userId) {
        // ...
    }
}

// 用 @Binds 绑定接口和实现（比 @Provides 更高效，不生成额外代码）
@Module
@InstallIn(SingletonComponent.class)
public abstract class RepositoryModule {

    @Binds
    @Singleton
    public abstract UserRepository bindUserRepository(UserRepositoryImpl impl);
}
```

**@Qualifier 区分同一类型的不同实例：**

```java
// 定义 Qualifier
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
public @interface AuthOkHttpClient {}

@Qualifier
@Retention(RetentionPolicy.RUNTIME)
public @interface NormalOkHttpClient {}

// Module 中提供两个不同的 OkHttpClient
@Module
@InstallIn(SingletonComponent.class)
public class NetworkModule {

    @Provides
    @Singleton
    @AuthOkHttpClient
    public OkHttpClient provideAuthOkHttpClient() {
        return new OkHttpClient.Builder()
            .addInterceptor(new AuthInterceptor())  // 带认证拦截器
            .build();
    }

    @Provides
    @Singleton
    @NormalOkHttpClient
    public OkHttpClient provideNormalOkHttpClient() {
        return new OkHttpClient.Builder()
            .build();  // 普通客户端
    }
}

// 使用时指定 Qualifier
public class AuthService {

    private final OkHttpClient client;

    @Inject
    public AuthService(@AuthOkHttpClient OkHttpClient client) {
        this.client = client;
    }
}
```

**@Named 替代 @Qualifier（简化写法）：**

```java
@Module
@InstallIn(SingletonComponent.class)
public class NetworkModule {

    @Provides
    @Singleton
    @Named("auth")
    public OkHttpClient provideAuthClient() { /* ... */ }

    @Provides
    @Singleton
    @Named("normal")
    public OkHttpClient provideNormalClient() { /* ... */ }
}

// 使用
@Inject
public AuthService(@Named("auth") OkHttpClient client) { /* ... */ }
```

#### 最佳实践

- ViewModel **必须**用 `@HiltViewModel`，不能用 `@AndroidEntryPoint`
- `@Module` 的 `@InstallIn` 决定了作用域，别什么都在 `SingletonComponent` 里
- 接口绑定优先用 `@Binds`（更高效），复杂对象用 `@Provides`
- 字段注入的变量不能是 `private`（Hilt 需要直接赋值）
- 测试时可以用 `@UninstallModules` 替换 Module，注入 Mock 对象

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| ViewModel 注入失败 | 用了 @AndroidEntryPoint 而不是 @HiltViewModel | ViewModel 用 @HiltViewModel |
| 编译报错找不到依赖 | Module 没有正确 InstallIn | 检查 @InstallIn 的组件 |
| 字段注入为 null | 字段是 private | Hilt 不支持 private 字段注入 |
| 编译慢 | Hilt 生成大量代码 | 可以考虑 Koin 做轻量替代 |

---

## 二、UI 类

---

### 8. ViewBinding

#### 一句话定位

自动为每个 XML 布局生成类型安全的绑定类，告别 `findViewById`。

#### 解决了什么问题

`findViewById` 有三大痛点：
1. 返回 `View`，你得手动强转类型，转错了运行时才崩
2. ID 写错了运行时才崩（返回 null，调方法就 NPE）
3. 每个 View 都得调一次，代码又长又丑

ViewBinding 编译时扫描 XML，生成绑定类，每个有 ID 的 View 都是**正确类型**的属性，空指针和类型转换错误在**编译时**就能发现。

#### 适用场景

- 所有使用 XML 布局的 Activity 和 Fragment
- 替代 `findViewById` 和 `kotlinx.android.synthetic`（已废弃）
- 需要 RecyclerView Adapter 中访问 Item 布局的 View

#### 核心原理

编译时 Gradle 插件扫描每个 XML 布局文件，根据文件名生成绑定类：
- `activity_main.xml` → `ActivityMainBinding`
- `fragment_profile.xml` → `FragmentProfileBinding`
- `item_user.xml` → `ItemUserBinding`

绑定类的 `inflate()` 内部调用 `LayoutInflater.inflate()`，然后对每个有 `android:id` 的 View 做 `findViewById` 并缓存到成员变量中。本质上是帮你写了 `findViewById` 的样板代码，但加了类型安全和空安全。

**ViewBinding vs DataBinding vs findViewById：**

| 特性 | findViewById | DataBinding | ViewBinding |
|------|-------------|-------------|-------------|
| 类型安全 | ❌ 需要强转 | ✅ | ✅ |
| 空安全 | ❌ 可能 NPE | ✅ | ✅ |
| 编译时检查 | ❌ | ✅ | ✅ |
| XML 中绑定数据 | ❌ | ✅ | ❌ |
| 性能 | 一般 | 较差（编译生成大量代码） | 好 |
| 使用复杂度 | 简单 | 复杂 | 简单 |

#### Gradle 配置

```groovy
android {
    buildFeatures {
        viewBinding = true
    }
}

// 如果只想为特定布局生成，在 XML 根节点加：
// tools:viewBindingIgnore="true"
```

#### 代码示例

**Activity 中使用：**

```java
public class MainActivity extends AppCompatActivity {

    private ActivityMainBinding binding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // inflate + setContentView 一步到位
        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());

        // 直接用，类型安全，不会 NPE
        binding.titleText.setText("Hello World");
        binding.subtitleText.setText("Jetpack ViewBinding");

        binding.submitButton.setOnClickListener(v -> {
            String input = binding.inputEditText.getText().toString();
            // 处理点击
        });

        binding.recyclerView.setLayoutManager(new LinearLayoutManager(this));
        binding.recyclerView.setAdapter(new MyAdapter());
    }
}
```

**Fragment 中使用（重点：必须在 onDestroyView 清理）：**

```java
public class ProfileFragment extends Fragment {

    private FragmentProfileBinding binding;

    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {
        binding = FragmentProfileBinding.inflate(inflater, container, false);
        return binding.getRoot();
    }

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        binding.nameText.setText("张三");
        binding.ageText.setText("25");

        binding.editButton.setOnClickListener(v -> {
            String name = binding.nameText.getText().toString();
            // 编辑操作
        });
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        binding = null;  // 必须置 null！Fragment 的视图可能比 Fragment 本身先销毁
    }
}
```

**RecyclerView Adapter 中使用：**

```java
public class UserAdapter extends RecyclerView.Adapter<UserAdapter.ViewHolder> {

    private List<User> users = new ArrayList<>();

    public void submitList(List<User> newUsers) {
        users = newUsers;
        notifyDataSetChanged();
    }

    @NonNull
    @Override
    public ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        ItemUserBinding binding = ItemUserBinding.inflate(
            LayoutInflater.from(parent.getContext()), parent, false);
        return new ViewHolder(binding);
    }

    @Override
    public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        holder.bind(users.get(position));
    }

    @Override
    public int getItemCount() {
        return users.size();
    }

    static class ViewHolder extends RecyclerView.ViewHolder {

        private final ItemUserBinding binding;

        ViewHolder(ItemUserBinding binding) {
            super(binding.getRoot());
            this.binding = binding;
        }

        void bind(User user) {
            binding.nameText.setText(user.getName());
            binding.emailText.setText(user.getEmail());
            binding.avatarImageView.setImageResource(user.getAvatarRes());
        }
    }
}
```

**Dialog 中使用：**

```java
public class ConfirmDialog extends Dialog {

    private DialogConfirmBinding binding;

    public ConfirmDialog(@NonNull Context context) {
        super(context);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        binding = DialogConfirmBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());

        binding.titleText.setText("确认操作");
        binding.messageText.setText("你确定要执行此操作吗？");
        binding.cancelButton.setOnClickListener(v -> dismiss());
        binding.confirmButton.setOnClickListener(v -> {
            // 确认操作
            dismiss();
        });
    }
}
```

#### 最佳实践

- Fragment 里**一定要**在 `onDestroyView` 中把 binding 置 null，否则内存泄漏（Fragment 的视图被销毁但 Fragment 对象还在，binding 持有视图引用）
- ViewBinding 比 DataBinding 轻量，不需要在 XML 里写 `<layout>` 和 `<data>` 标签
- 如果不需要在 XML 里绑定数据，就用 ViewBinding，别用 DataBinding
- Adapter 的 ViewHolder 中持有 binding 引用，比直接持有 View 更清晰

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| Fragment 内存泄漏 | onDestroyView 没置 null | binding = null |
| 找不到绑定类 | ViewBinding 没开启 | buildFeatures { viewBinding = true } |
| 某些布局不想生成绑定类 | 不需要 | XML 根节点加 tools:viewBindingIgnore="true" |

---

### 9. Fragment

#### 一句话定位

Activity 里的"可插拔模块"，有自己的生命周期和布局，可以复用和组合。

#### 解决了什么问题

一个页面可能很复杂（比如一个页面同时显示列表和详情），全写在一个 Activity 里代码量爆炸。Fragment 把页面拆成多个独立模块，每个模块有自己的布局和逻辑，可以复用（比如同一个 Fragment 在手机上全屏、在平板上并排显示）。

#### 适用场景

- 页面模块化（一个页面由多个 Fragment 组成）
- 多面板布局（手机单列、平板双列）
- 底部导航栏切换页面
- ViewPager + Fragment 实现滑动切换
- Dialog（DialogFragment）
- Fragment 间通信（共享 ViewModel）

#### 核心原理

Fragment 不能独立存在，必须住在 Activity 里。它有自己的生命周期（比 Activity 多了 `onCreateView` 和 `onDestroyView`），视图可以独立创建和销毁。`FragmentManager` 管理所有 Fragment 的事务和返回栈。

**Fragment 生命周期（比 Activity 多了两步）：**

```
Activity:    onCreate → onResume → onPause → onDestroy
Fragment:    onAttach → onCreate → onCreateView → onViewCreated →
             onStart → onResume → onPause → onStop →
             onDestroyView → onDestroy → onDetach
```

关键区别：
- `onDestroyView`：Fragment 的视图被销毁（比如被 replace 了），但 Fragment 对象还在
- `onDestroy`：Fragment 对象被销毁
- 这就是为什么 Fragment 中观察 LiveData 要用 `getViewLifecycleOwner()` 而不是 `this`

**Fragment 事务：**

```java
FragmentManager fragmentManager = getSupportFragmentManager();
FragmentTransaction transaction = fragmentManager.beginTransaction();

transaction.replace(R.id.container, new ProfileFragment());
transaction.addToBackStack("profile");  // 加入返回栈，按返回键可以回来
transaction.setReorderingAllowed(true); // 优化事务执行顺序
transaction.commit();                   // 提交（异步）
// transaction.commitAllowingStateLoss(); // 允许状态丢失（不推荐）
```

#### 关键 API

| API | 用途 | 说明 |
|-----|------|------|
| `Fragment(requireContext())` | 构造函数 | 传入布局 ID |
| `onCreateView()` | 创建视图 | 返回 Fragment 的根视图 |
| `onViewCreated()` | 视图创建完成 | 在这里初始化 View 和绑定数据 |
| `getViewLifecycleOwner()` | 视图生命周期 | 观察 LiveData 时用这个 |
| `requireContext()` | 获取 Context | 非 null 版本的 getContext() |
| `requireActivity()` | 获取宿主 Activity | 非 null 版本的 getActivity() |
| `requireArguments()` | 获取参数 | 非 null 版本的 getArguments() |
| `setArguments(bundle)` | 设置参数 | 在创建时传入 |
| `setResult(result)` | 设置返回结果 | Fragment Result API |
| `getParentFragmentManager()` | 获取父 FragmentManager | 嵌套 Fragment 时用 |
| `setReorderingAllowed(true)` | 优化事务 | 推荐开启 |

#### 代码示例

**基础用法：**

```java
public class ProfileFragment extends Fragment {

    private FragmentProfileBinding binding;
    private ProfileViewModel viewModel;

    // 推荐用工厂方法创建 Fragment 并传参
    public static ProfileFragment newInstance(String userId) {
        ProfileFragment fragment = new ProfileFragment();
        Bundle args = new Bundle();
        args.putString("user_id", userId);
        fragment.setArguments(args);
        return fragment;
    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 获取参数
        String userId = requireArguments().getString("user_id");
        // 创建 ViewModel（注意：这里不能用 getViewLifecycleOwner，因为视图还没创建）
        viewModel = new ViewModelProvider(this).get(ProfileViewModel.class);
        viewModel.setUserId(userId);
    }

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {
        binding = FragmentProfileBinding.inflate(inflater, container, false);
        return binding.getRoot();
    }

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        // 观察 LiveData（必须用 getViewLifecycleOwner()）
        viewModel.getUser().observe(getViewLifecycleOwner(), user -> {
            binding.nameText.setText(user.getName());
            binding.emailText.setText(user.getEmail());
        });

        viewModel.getIsLoading().observe(getViewLifecycleOwner(), loading -> {
            binding.progressBar.setVisibility(loading ? View.VISIBLE : View.GONE);
        });

        binding.editButton.setOnClickListener(v -> {
            // 导航到编辑页
            Navigation.findNavController(v).navigate(
                R.id.action_profile_to_edit
            );
        });
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        binding = null;
    }
}
```

**Fragment 间通信（Fragment Result API）：**

```java
// 发送结果的 Fragment
public class InputFragment extends Fragment {

    private void sendResult(String inputValue) {
        Bundle result = new Bundle();
        result.putString("input_value", inputValue);
        // 用 setFragmentResult 发送，key 是请求 key
        getParentFragmentManager().setFragmentResult("request_key", result);
        // Navigation.findNavController(requireView()).popBackStack(); // 返回上一页
    }
}

// 接收结果的 Fragment（或 Activity）
public class ResultFragment extends Fragment {

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 注册结果监听
        getParentFragmentManager().setFragmentResultListener(
            "request_key",
            this,
            (requestKey, result) -> {
                String value = result.getString("input_value");
                // 处理结果
                binding.resultText.setText(value);
            }
        );
    }
}
```

**嵌套 Fragment（子 Fragment）：**

```java
// 父 Fragment 的布局中包含一个 FrameLayout 作为子 Fragment 容器
public class ParentFragment extends Fragment {

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);

        // 用 getChildFragmentManager() 管理子 Fragment
        getChildFragmentManager().beginTransaction()
            .replace(R.id.child_container, new ChildFragment())
            .commit();
    }
}
```

#### 最佳实践

- Fragment 里观察 LiveData/Flow **必须**用 `getViewLifecycleOwner()`，别用 `this`。因为 Fragment 的生命周期比视图长（replace 时视图销毁但 Fragment 还在），用 `this` 会导致重复观察或崩溃
- 传参用 `newInstance()` 工厂方法 + `Bundle`，别用构造函数传参（系统重建 Fragment 时会用无参构造函数）
- `setReorderingAllowed(true)` 可以优化事务执行顺序，推荐开启
- Fragment 间通信用**共享 ViewModel** 或 **Fragment Result API**，别直接互相引用
- 嵌套 Fragment 用 `getChildFragmentManager()`，别用 `getParentFragmentManager()`

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| LiveData 收到多次回调 | 用了 this 而不是 getViewLifecycleOwner() | 改用 getViewLifecycleOwner() |
| 参数丢失 | 用了构造函数传参 | 用 newInstance() + Bundle |
| 嵌套 Fragment 返回栈混乱 | 用了错误的 FragmentManager | 子 Fragment 用 getChildFragmentManager() |
| IllegalStateException: Fragment already added | 重复添加同一个 Fragment | 检查是否已添加，或用 replace |

---

### 10. RecyclerView（ListAdapter + DiffUtil）

#### 一句话定位

高效列表控件，通过复用 Item View 来显示大量数据，配合 DiffUtil 只更新变化的项。

#### 解决了什么问题

`ListView` 每次滚动都创建新 View，数据多了就卡。RecyclerView 通过 ViewHolder 模式复用已滚出屏幕的 View，性能好很多。而且 `DiffUtil` 可以自动计算列表差异，只更新变化的 Item，带平滑动画。

#### 适用场景

- 任何需要显示列表数据的场景（用户列表、商品列表、消息列表等）
- 需要不同布局（线性、网格、瀑布流）
- 需要列表项动画（增删改的过渡动画）
- 大量数据展示（配合 Paging 分页加载）

#### 核心原理

打个比方：RecyclerView 就像自助餐厅。它只有固定数量的盘子（ViewHolder），吃完的盘子收回来洗干净给下一个人用（复用），而不是每个人发一个新盘子。这样不管来多少人，盘子数量始终可控。

技术层面：

- **ViewHolder 模式**：每个 Item 对应一个 ViewHolder，滚出屏幕的 ViewHolder 被回收放入 Pool，新 Item 进入时从 Pool 中取出复用
- **LayoutManager**：决定怎么排列 Item
  - `LinearLayoutManager`：线性列表（水平/垂直）
  - `GridLayoutManager`：网格布局
  - `StaggeredGridLayoutManager`：瀑布流
- **ItemDecoration**：添加分隔线、边距
- **ItemAnimator**：控制增删改的动画（默认有 `DefaultItemAnimator`）
- **DiffUtil**：用 Eugene W. Myers 差分算法计算新旧列表差异，时间复杂度 O(N)，只更新变化的 Item
- **ListAdapter**：内置 `AsyncListDiffer` 的适配器，调用 `submitList()` 自动在后台线程计算差异并更新

**RecyclerView 的回收复用流程：**

```
Item 滚出屏幕
  → RecyclerView 回收 ViewHolder，放入 Scrap（临时缓存）或 Cache（一级缓存）
  → 新 Item 进入屏幕
  → 先从 Scrap 找（同一帧内的复用）
  → 再从 Cache 找（刚滚出去的，带完整状态）
  → 再从 ViewCacheExtension 找（自定义缓存）
  → 再从 RecycledViewPool 找（按 viewType 找，需要重新绑定数据）
  → 都没有才创建新的 ViewHolder
```

#### 关键 API

| API | 用途 | 说明 |
|-----|------|------|
| `RecyclerView` | 核心控件 | 列表容器 |
| `RecyclerView.ViewHolder` | 视图持有者 | 持有一个 Item 的视图 |
| `RecyclerView.Adapter<VH>` | 适配器基类 | 管理 ViewHolder 的创建和绑定 |
| `ListAdapter<T, VH>` | 内置 DiffUtil 的适配器 | 推荐，自动计算差异 |
| `DiffUtil.ItemCallback<T>` | 差异计算回调 | 定义"什么是同一个 Item"和"内容是否变化" |
| `AsyncListDiffer` | 异步差异计算器 | 自定义 Adapter 时使用 |
| `LinearLayoutManager` | 线性布局 | 最常用的布局管理器 |
| `GridLayoutManager` | 网格布局 | 可设置列数 |
| `ItemDecoration` | 分隔线 | 添加 Item 之间的间距或分隔线 |
| `submitList(list)` | 提交新数据 | ListAdapter 的方法，自动计算差异 |

#### 代码示例

**完整用法（ListAdapter + ViewBinding）：**

```java
// 1. 定义数据类
public class User {
    private final String id;
    private final String name;
    private final String email;
    private final int avatarRes;

    public User(String id, String name, String email, int avatarRes) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.avatarRes = avatarRes;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public int getAvatarRes() { return avatarRes; }
}

// 2. 定义 DiffUtil 回调
public class UserDiffCallback extends DiffUtil.ItemCallback<User> {

    @Override
    public boolean areItemsTheSame(@NonNull User oldItem, @NonNull User newItem) {
        // 判断是否是同一个 Item（通常用唯一标识）
        return oldItem.getId().equals(newItem.getId());
    }

    @Override
    public boolean areContentsTheSame(@NonNull User oldItem, @NonNull User newItem) {
        // 判断内容是否变化（通常用 equals）
        return oldItem.equals(newItem);
    }

    // 可选：定义部分更新（只更新变化的字段）
    @Nullable
    @Override
    public Object getChangePayload(@NonNull User oldItem, @NonNull User newItem) {
        Bundle diff = new Bundle();
        if (!oldItem.getName().equals(newItem.getName())) {
            diff.putString("name", newItem.getName());
        }
        if (!oldItem.getEmail().equals(newItem.getEmail())) {
            diff.putString("email", newItem.getEmail());
        }
        return diff.isEmpty() ? null : diff;
    }
}

// 3. 定义 Adapter
public class UserAdapter extends ListAdapter<User, UserAdapter.ViewHolder> {

    private final OnItemClickListener listener;

    public UserAdapter(OnItemClickListener listener) {
        super(new UserDiffCallback());
        this.listener = listener;
    }

    @NonNull
    @Override
    public ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        ItemUserBinding binding = ItemUserBinding.inflate(
            LayoutInflater.from(parent.getContext()), parent, false);
        return new ViewHolder(binding);
    }

    @Override
    public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        holder.bind(getItem(position));
    }

    @Override
    public void onBindViewHolder(@NonNull ViewHolder holder, int position,
                                @NonNull List<Object> payloads) {
        // 部分更新：payloads 不为空时只更新变化的字段
        if (payloads.isEmpty()) {
            onBindViewHolder(holder, position);
        } else {
            Bundle diff = (Bundle) payloads.get(0);
            if (diff.containsKey("name")) {
                holder.binding.nameText.setText(diff.getString("name"));
            }
            if (diff.containsKey("email")) {
                holder.binding.emailText.setText(diff.getString("email"));
            }
        }
    }

    // 点击事件接口
    public interface OnItemClickListener {
        void onItemClick(User user);
        void onLongClick(User user);
    }

    class ViewHolder extends RecyclerView.ViewHolder {

        private final ItemUserBinding binding;

        ViewHolder(ItemUserBinding binding) {
            super(binding.getRoot());
            this.binding = binding;
        }

        void bind(User user) {
            binding.nameText.setText(user.getName());
            binding.emailText.setText(user.getEmail());
            binding.avatarImageView.setImageResource(user.getAvatarRes());

            binding.getRoot().setOnClickListener(v -> listener.onItemClick(user));
            binding.getRoot().setOnLongClickListener(v -> {
                listener.onLongClick(user);
                return true;
            });
        }
    }
}

// 4. 在 Activity/Fragment 中使用
public class UserListFragment extends Fragment {

    private FragmentUserListBinding binding;
    private UserAdapter adapter;

    @Override
    public void onViewCreated(@NonNull View view, @Nullable Bundle savedInstanceState) {
        super.onViewCreated(view, savedInstanceState);
        binding = FragmentUserListBinding.bind(view);

        // 创建 Adapter
        adapter = new UserAdapter(new UserAdapter.OnItemClickListener() {
            @Override
            public void onItemClick(User user) {
                // 跳转到详情
                Bundle args = new Bundle();
                args.putString("user_id", user.getId());
                Navigation.findNavController(requireView()).navigate(
                    R.id.action_list_to_detail, args);
            }

            @Override
            public void onLongClick(User user) {
                // 长按操作（删除、编辑等）
                new AlertDialog.Builder(requireContext())
                    .setTitle("操作")
                    .setItems(new String[]{"编辑", "删除"}, (dialog, which) -> {
                        if (which == 0) { /* 编辑 */ }
                        else { /* 删除 */ }
                    })
                    .show();
            }
        });

        // 配置 RecyclerView
        binding.recyclerView.setLayoutManager(new LinearLayoutManager(requireContext()));
        binding.recyclerView.setAdapter(adapter);

        // 添加分隔线
        binding.recyclerView.addItemDecoration(
            new DividerItemDecoration(requireContext(), DividerItemDecoration.VERTICAL));

        // 提交数据（自动计算差异并更新）
        adapter.submitList(userList);

        // 观察数据变化
        viewModel.getUsers().observe(getViewLifecycleOwner(), users -> {
            adapter.submitList(users);
        });
    }
}
```

**多类型 Item（不同布局）：**

```java
public class MessageAdapter extends ListAdapter<MessageItem, RecyclerView.ViewHolder> {

    private static final int TYPE_TEXT = 0;
    private static final int TYPE_IMAGE = 1;
    private static final int TYPE_DATE = 2;

    @Override
    public int getItemViewType(int position) {
        MessageItem item = getItem(position);
        if (item instanceof TextMessage) return TYPE_TEXT;
        if (item instanceof ImageMessage) return TYPE_IMAGE;
        if (item instanceof DateDivider) return TYPE_DATE;
        return TYPE_TEXT;
    }

    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        switch (viewType) {
            case TYPE_IMAGE:
                return new ImageViewHolder(ItemMessageImageBinding.inflate(
                    LayoutInflater.from(parent.getContext()), parent, false));
            case TYPE_DATE:
                return new DateViewHolder(ItemDateDividerBinding.inflate(
                    LayoutInflater.from(parent.getContext()), parent, false));
            default:
                return new TextViewHolder(ItemMessageTextBinding.inflate(
                    LayoutInflater.from(parent.getContext()), parent, false));
        }
    }

    @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position) {
        MessageItem item = getItem(position);
        if (holder instanceof TextViewHolder) {
            ((TextViewHolder) holder).bind((TextMessage) item);
        } else if (holder instanceof ImageViewHolder) {
            ((ImageViewHolder) holder).bind((ImageMessage) item);
        } else if (holder instanceof DateViewHolder) {
            ((DateViewHolder) holder).bind((DateDivider) item);
        }
    }
}
```

**自定义 ItemDecoration（分隔线）：**

```java
public class SpacingItemDecoration extends RecyclerView.ItemDecoration {

    private final int verticalSpacing;
    private final int horizontalSpacing;

    public SpacingItemDecoration(int verticalSpacing, int horizontalSpacing) {
        this.verticalSpacing = verticalSpacing;
        this.horizontalSpacing = horizontalSpacing;
    }

    @Override
    public void getItemOffsets(@NonNull Rect outRect, @NonNull View view,
                               @NonNull RecyclerView parent,
                               @NonNull RecyclerView.State state) {
        int position = parent.getChildAdapterPosition(view);
        int itemCount = state.getItemCount();

        // 最后一项不加底部间距
        outRect.bottom = (position == itemCount - 1) ? 0 : verticalSpacing;
        outRect.left = horizontalSpacing;
        outRect.right = horizontalSpacing;
        outRect.top = verticalSpacing;
    }
}

// 使用
recyclerView.addItemDecoration(new SpacingItemDecoration(
    getResources().getDimensionPixelSize(R.dimen.spacing_8dp),
    getResources().getDimensionPixelSize(R.dimen.spacing_16dp)
));
```

#### 最佳实践

- **永远用 `ListAdapter` + `submitList()`**，别用 `notifyDataSetChanged()`（全量刷新，没有动画，性能差）
- `submitList()` 传入的列表必须是**新的对象**，传同一个对象不会触发更新（内部用 `==` 判断）
- `areItemsTheSame` 用唯一标识（ID），`areContentsTheSame` 用 `equals()`
- 列表项用 `holder.itemView.setTag()` 或 `RecyclerView.ViewHolder.getItemId()` 帮助定位
- 大量数据配合 Paging 使用，别一次性加载全部
- `RecyclerView.setHasFixedSize(true)` 在 Item 尺寸固定时开启，可以跳过 `requestLayout` 提升性能

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 数据没更新 | submitList 传了同一个对象 | 创建新列表对象 |
| 动画不流畅 | DiffUtil 计算慢 | 确保 areItemsTheSame 正确 |
| Item 闪烁 | onBindViewHolder 中做了耗时操作 | 确保绑定操作快速 |
| 嵌套滚动冲突 | RecyclerView 在 ScrollView 中 | 用 NestedScrollView 或设置 isNestedScrollingEnabled |

---

## 三、基础类（Foundation）

---

### 11. WorkManager

#### 一句话定位

后台任务的"保险箱"，就算 App 被杀了、手机重启了，任务照样执行。

#### 解决了什么问题

后台任务（上传日志、同步数据、定期清理缓存）需要可靠执行，但 Android 的后台限制越来越严：
- Android 6.0+ 的 Doze 模式会限制后台网络和 CPU
- Android 8.0+ 的后台执行限制禁止隐式广播
- Android 12+ 的前台服务启动限制更严了

WorkManager 帮你选最合适的执行方式（API 23+ 用 JobScheduler，低版本用 AlarmManager + BroadcastReceiver），保证任务一定能跑完。

#### 适用场景

- **必须执行的后台任务**（上传数据、同步、发送日志）
- **不需要立即执行的任务**（可以延迟几分钟甚至几小时）
- **需要条件触发**（有 WiFi 时上传、充电时同步）
- **周期性任务**（每天凌晨清理缓存、每小时同步一次）
- **任务链**（先压缩再上传，上传成功再通知）

**不适合的场景：**
- 需要立即执行的任务 → 用 Foreground Service
- 精确定时的任务 → 用 AlarmManager
- 短时间内的后台任务 → 用 Coroutine 或 Executor

#### 核心原理

打个比方：WorkManager 就像一个靠谱的快递公司。你把包裹（任务）交给它，它会在合适的时间送达。就算路上遇到暴风雨（App 被杀、手机重启），它也会在条件恢复后继续送。你还可以设定条件——"等有 WiFi 再送"（Constraints）。

技术层面：

- 内部用**自带的 SQLite 数据库**持久化所有任务（WorkInfo），设备重启后自动恢复未完成的任务
- API 23+ 用 `JobScheduler`，低版本用 `AlarmManager + BroadcastReceiver`，自动选择最优方案
- 支持约束条件（Constraints）：网络类型、充电状态、电量低、存储空间
- 支持重试策略：指数退避（Exponential Backoff），可配置最大重试次数
- 支持任务链（WorkChain）：顺序执行、并行执行、合并执行
- 加急工作（Expedited Work）：可以立即执行，类似 Foreground Service（API 12+）
- 任务状态机：`ENQUEUED → RUNNING → SUCCEEDED / FAILED / RETRYING / CANCELLED / BLOCKED`

**WorkManager 任务状态流转：**

```
ENQUEUED（已入队，等待条件满足）
  ↓
RUNNING（正在执行）
  ↓
SUCCEEDED（成功） / FAILED（失败） / RETRYING（重试中）
  ↓
CANCELLED（被取消）
```

#### 关键 API

| API | 用途 | 说明 |
|-----|------|------|
| `Worker` | 任务基类 | 同步执行，doWork() 在后台线程 |
| `CoroutineWorker` | 协程任务 | Kotlin 用（Java 用 ListenableWorker） |
| `RxWorker` | RxJava 任务 | Java 推荐，支持 RxJava |
| `OneTimeWorkRequest` | 一次性任务 | 执行一次就结束 |
| `PeriodicWorkRequest` | 周期性任务 | 最小间隔 15 分钟 |
| `WorkRequest.Builder` | 构建器 | 设置约束、标签、初始延迟等 |
| `Constraints` | 约束条件 | 网络、充电、电量等 |
| `WorkManager.enqueue()` | 调度任务 | 提交任务到队列 |
| `WorkManager.beginWith().then()` | 任务链 | 顺序/并行组合 |
| `WorkManager.getWorkInfoById()` | 查询状态 | 返回 LiveData |
| `WorkManager.cancelWorkById()` | 取消任务 | 按任务 ID 取消 |
| `setExpedited()` | 加急执行 | 立即执行（API 12+） |
| `setBackoffCriteria()` | 重试策略 | 指数退避配置 |
| `setInputData()` / `getInputData()` | 传递数据 | 任务间传参 |

#### Gradle 依赖

```groovy
dependencies {
    implementation "androidx.work:work-runtime:2.10.0"
    // RxJava 支持（Java 推荐）
    implementation "androidx.work:work-rxjava3:2.10.0"
}
```

#### 代码示例

**基础用法（一次性任务）：**

```java
// 1. 定义 Worker
public class UploadWorker extends Worker {

    public UploadWorker(@NonNull Context context, @NonNull WorkerParameters params) {
        super(context, params);
    }

    @NonNull
    @Override
    public Result doWork() {
        // doWork() 已经在后台线程执行了
        try {
            String filePath = getInputData().getString("file_path");
            File file = new File(filePath);

            // 模拟上传
            boolean success = uploadFile(file);

            if (success) {
                return Result.success();  // 成功
            } else {
                return Result.failure();  // 失败
            }
        } catch (Exception e) {
            // 返回 retry 会触发重试（按 BackoffCriteria 配置的策略）
            if (getRunAttemptCount() < 3) {
                return Result.retry();
            }
            return Result.failure();
        }
    }

    private boolean uploadFile(File file) {
        // 实际上传逻辑
        return true;
    }
}

// 2. 构建并调度
public class UploadHelper {

    public static void scheduleUpload(Context context, String filePath) {
        // 传递数据给 Worker
        Data inputData = new Data.Builder()
            .putString("file_path", filePath)
            .build();

        // 设置约束条件
        Constraints constraints = new Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)  // 需要网络
            .setRequiresBatteryNotLow(true)                 // 电量不低
            .build();

        // 构建任务请求
        OneTimeWorkRequest uploadWork = new OneTimeWorkRequest.Builder(UploadWorker.class)
            .setInputData(inputData)
            .setConstraints(constraints)
            .setBackoffCriteria(                          // 重试策略
                BackoffPolicy.EXPONENTIAL,                // 指数退避
                10, TimeUnit.MINUTES,                     // 初始延迟 10 分钟
                30, TimeUnit.MINUTES                      // 最大延迟 30 分钟
            )
            .addTag("upload")                             // 标签，方便查询和取消
            .setInitialDelay(5, TimeUnit.SECONDS)         // 初始延迟 5 秒
            .build();

        // 提交任务
        WorkManager.getInstance(context).enqueue(uploadWork);
    }
}

// 3. 观察任务状态
public class UploadActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        OneTimeWorkRequest uploadWork = UploadHelper.buildUploadWork("/path/to/file");

        // 用 LiveData 观察状态
        WorkManager.getInstance(this).getWorkInfoByIdLiveData(uploadWork.getId())
            .observe(this, workInfo -> {
                if (workInfo == null) return;

                switch (workInfo.getState()) {
                    case ENQUEUED:
                        showToast("任务已入队，等待执行");
                        break;
                    case RUNNING:
                        showToast("正在上传...");
                        progressBar.setVisibility(View.VISIBLE);
                        break;
                    case SUCCEEDED:
                        showToast("上传成功");
                        progressBar.setVisibility(View.GONE);
                        break;
                    case FAILED:
                        showToast("上传失败");
                        progressBar.setVisibility(View.GONE);
                        break;
                    case RETRYING:
                        showToast("上传失败，正在重试...");
                        break;
                    case CANCELLED:
                        showToast("任务已取消");
                        break;
                }
            });

        WorkManager.getInstance(this).enqueue(uploadWork);
    }
}
```

**RxWorker（Java 推荐，支持异步操作）：**

```java
public class SyncWorker extends RxWorker {

    public SyncWorker(@NonNull Context context, @NonNull WorkerParameters params) {
        super(context, params);
    }

    @NonNull
    @Override
    public Single<Result> createWork() {
        return apiService.syncData()
            .map(response -> {
                if (response.isSuccess()) {
                    return Result.success(
                        new Data.Builder()
                            .putString("sync_count", String.valueOf(response.getCount()))
                            .build()
                    );
                } else {
                    return Result.retry();
                }
            })
            .onErrorReturnItem(Result.failure());
    }
}
```

**任务链（顺序 + 并行）：**

```java
// 场景：先压缩图片，再上传，上传成功后发通知
// 两个图片可以并行压缩，然后上传，最后发通知

OneTimeWorkRequest compressA = new OneTimeWorkRequest.Builder(CompressWorker.class)
    .setInputData(new Data.Builder().putString("file", "image_a.jpg").build())
    .build();

OneTimeWorkRequest compressB = new OneTimeWorkRequest.Builder(CompressWorker.class)
    .setInputData(new Data.Builder().putString("file", "image_b.jpg").build())
    .build();

OneTimeWorkRequest upload = new OneTimeWorkRequest.Builder(UploadWorker.class)
    .build();

OneTimeWorkRequest notify = new OneTimeWorkRequest.Builder(NotifyWorker.class)
    .build();

// 并行压缩 → 顺序上传 → 发通知
WorkManager.getInstance(context)
    .beginWith(Arrays.asList(compressA, compressB))  // 并行执行
    .then(upload)                                     // 顺序执行
    .then(notify)                                     // 最后发通知
    .enqueue();

// 观察整个链的状态
WorkManager.getInstance(context).getWorkInfoByIdLiveData(notify.getId())
    .observe(this, workInfo -> {
        if (workInfo != null && workInfo.getState() == WorkInfo.State.SUCCEEDED) {
            showToast("全部完成！");
        }
    });
```

**周期性任务：**

```java
// 每天凌晨 3 点清理缓存（最小间隔 15 分钟，重复间隔 24 小时）
PeriodicWorkRequest cleanupWork = new PeriodicWorkRequest.Builder(
    CleanupWorker.class,
    24, TimeUnit.HOURS,           // 重复间隔
    1, TimeUnit.HOURS             // flex interval（在此窗口内执行）
)
    .setConstraints(new Constraints.Builder()
        .setRequiresDeviceIdle(true)  // 设备空闲时执行
        .build())
    .build();

WorkManager.getInstance(context)
    .enqueueUniquePeriodicWork(
        "daily_cleanup",                        // 唯一名称
        ExistingPeriodicWorkPolicy.KEEP,        // 已存在则保留（不替换）
        cleanupWork
    );
```

**取消任务：**

```java
// 按标签取消所有任务
WorkManager.getInstance(context).cancelAllWorkByTag("upload");

// 按 ID 取消
WorkManager.getInstance(context).cancelWorkById(workRequest.getId());

// 按唯一名称取消
WorkManager.getInstance(context).cancelUniqueWork("daily_cleanup");
```

#### 最佳实践

- 不需要保证立即执行的任务用 WorkManager，需要立即执行的用 Foreground Service
- 周期任务最小间隔 **15 分钟**（系统限制），别设更小
- Worker 的 `doWork()` 不要做超过 **10 分钟**的操作，会被系统杀掉
- 用 `enqueueUniqueWork()` + `ExistingWorkPolicy.REPLACE` 避免重复入队同一任务
- 传大数据给 Worker 别用 `Data`（有 10KB 限制），存到数据库/文件里传 URI
- RxWorker 是 Java 项目的最佳选择，天然支持异步操作

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 任务不执行 | 约束条件不满足 | 检查 Constraints 和设备状态 |
| 任务重复执行 | 没用 enqueueUniqueWork | 用 enqueueUniqueWork + REPLACE |
| 传大数据失败 | Data 有 10KB 限制 | 存文件/数据库，传 URI |
| 任务被系统杀 | 执行时间太长 | 拆分任务或用 setForeground |

---

### 12. App Startup

#### 一句话定位

统一管理 App 启动时的初始化，替代每个库各自用 ContentProvider 的做法。

#### 解决了什么问题

以前每个第三方库（WorkManager、Firebase、LeakCanary 等）启动时都自己创建一个 ContentProvider 来初始化。App 启动时系统会按顺序创建所有 ContentProvider，每个都要花时间，库多了启动就慢。

App Startup 把所有初始化合并到一个 ContentProvider 里，按依赖顺序执行，显著减少启动时间。

#### 适用场景

- App 启动时需要初始化多个第三方库
- 想控制初始化顺序（A 必须在 B 之前初始化）
- 想延迟某些库的初始化（懒加载）
- 作为库的开发者，提供初始化入口

#### 核心原理

打个比方：以前每个库启动时都自己开一扇门（ContentProvider）进你的 App，门多了启动就慢。App Startup 把所有门合成一扇——一个 ContentProvider 统一按顺序初始化所有库。

技术层面：

- 每个库实现 `Initializer<T>` 接口，声明 `create(context)` 和 `dependencies()`
- App Startup 在 `Application.onCreate()` **之前**通过 `InitializationProvider`（一个 ContentProvider）执行
- 按依赖关系拓扑排序，依次初始化（没有依赖的先初始化）
- 也可以禁用自动初始化，手动调用 `AppInitializer.getInstance().initializeComponent()`

**初始化顺序示例：**

```
LoggerInitializer（无依赖）→ 先初始化
  └── WorkManagerInitializer（依赖 Logger）→ 后初始化
        └── AnalyticsInitializer（依赖 WorkManager）→ 最后初始化
```

#### 关键 API

| API | 用途 | 说明 |
|-----|------|------|
| `Initializer<T>` | 初始化器接口 | 实现 create() 和 dependencies() |
| `create(context)` | 执行初始化 | 返回初始化后的对象 |
| `dependencies()` | 声明依赖 | 返回依赖的其他 Initializer 的 Class 列表 |
| `AppInitializer` | 手动初始化入口 | 手动触发某个 Initializer |
| `InitializationProvider` | 内置 ContentProvider | 自动发现并执行 Initializer |

#### Gradle 依赖

```groovy
implementation "androidx.startup:startup-runtime:1.2.0"
```

#### 代码示例

**定义初始化器：**

```java
// 1. 日志库初始化（无依赖，最先执行）
public class LoggerInitializer implements Initializer<Logger> {

    @NonNull
    @Override
    public Logger create(@NonNull Context context) {
        Logger.init(context)
            .setLogLevel(BuildConfig.DEBUG ? LogLevel.DEBUG : LogLevel.NONE)
            .enableFileLogging(true);
        return Logger.getInstance();
    }

    @NonNull
    @Override
    public List<Class<? extends Initializer<?>>> dependencies() {
        return Collections.emptyList(); // 无依赖
    }
}

// 2. WorkManager 初始化（依赖 Logger）
public class WorkManagerInitializer implements Initializer<WorkManager> {

    @NonNull
    @Override
    public WorkManager create(@NonNull Context context) {
        Configuration config = new Configuration.Builder()
            .setMinimumLoggingLevel(Log.DEBUG)
            .build();
        WorkManager.initialize(context, config);
        return WorkManager.getInstance(context);
    }

    @NonNull
    @Override
    public List<Class<? extends Initializer<?>>> dependencies() {
        return Collections.singletonList(LoggerInitializer.class); // 依赖 Logger
    }
}

// 3. 分析库初始化（依赖 WorkManager）
public class AnalyticsInitializer implements Initializer<AnalyticsService> {

    @NonNull
    @Override
    public AnalyticsService create(@NonNull Context context) {
        AnalyticsService service = new AnalyticsService(context);
        service.start();
        return service;
    }

    @NonNull
    @Override
    public List<Class<? extends Initializer<?>>> dependencies() {
        return Collections.singletonList(WorkManagerInitializer.class); // 依赖 WorkManager
    }
}
```

**在 AndroidManifest.xml 中注册：**

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">

    <!-- 注册初始化器 -->
    <meta-data
        android:name="com.example.app.LoggerInitializer"
        android:value="androidx.startup" />

    <meta-data
        android:name="com.example.app.WorkManagerInitializer"
        android:value="androidx.startup" />

    <meta-data
        android:name="com.example.app.AnalyticsInitializer"
        android:value="androidx.startup" />

</provider>
```

**禁用自动初始化（延迟/手动初始化）：**

```xml
<!-- 禁用某个库的自动初始化 -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">

    <!-- 用 tools:node="remove" 移除自动初始化 -->
    <meta-data
        android:name="com.example.app.AnalyticsInitializer"
        android:value="androidx.startup"
        tools:node="remove" />

</provider>
```

```java
// 手动初始化（在需要的时候）
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // 手动触发初始化
        AppInitializer.getInstance(this)
            .initializeComponent(AnalyticsInitializer.class);
    }
}
```

#### 最佳实践

- 第三方库（如 WorkManager、Firebase）已经自带了 Startup 初始化器，不需要你手动写
- 如果想延迟初始化某个库，用 `tools:node="remove"` 禁用自动初始化，在需要时手动调用
- Startup 只解决"启动时初始化"的问题，懒加载还得自己处理
- 初始化器不要做耗时操作，否则影响启动速度

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 初始化顺序不对 | dependencies() 没正确声明 | 检查依赖关系 |
| 初始化器没执行 | Manifest 里没注册 | 添加 meta-data |
| 想延迟初始化 | 自动初始化太早 | 用 tools:node="remove" 禁用 |

---

### 13. SplashScreen

#### 一句话定位

统一的启动画面 API，Android 12+ 是系统行为，低版本兼容库帮你搞定。

#### 解决了什么问题

以前启动画面（Splash Screen）需要自己实现一个 Activity 或用第三方库，不同设备表现不一致。Android 12 开始系统强制显示启动画面（带 App 图标和动画），你没法关掉只能自定义。`core-splashscreen` 兼容库把这个行为统一到 Android 6.0+。

#### 适用场景

- 所有 App 都需要启动画面（Android 12+ 强制要求）
- 需要自定义启动画面的图标、背景色、动画
- 需要控制启动画面的显示时长（等数据加载完再关闭）
- 需要自定义退出动画

#### 核心原理

Android 12 开始系统强制显示启动画面。`core-splashscreen` 兼容库把这个行为向后兼容到 Android 6.0+。

启动画面分三个阶段：

1. **进入动画**（系统控制）：冷启动时系统显示，图标从中心放大到目标位置
2. **等待阶段**（你控制）：App 在后台初始化，画面保持显示。你可以用 `setKeepOnScreenCondition` 控制何时关闭
3. **退出动画**（你控制）：初始化完成，画面消失。你可以自定义退出动画（淡出、缩放等）

**时间线：**

```
系统启动 → 进入动画（系统控制）→ 等待阶段（你控制）→ 退出动画（你控制）→ 第一帧绘制
```

#### 关键 API

| API | 用途 | 说明 |
|-----|------|------|
| `installSplashScreen()` | 安装启动画面 | 在 Activity 的 onCreate 中调用，super.onCreate() 之前 |
| `setKeepOnScreenCondition()` | 控制显示时长 | 返回 true 继续显示，false 关闭 |
| `setOnExitAnimationListener()` | 自定义退出动画 | 监听退出事件，自己控制动画 |
| `SplashScreenView` | 启动画面视图 | 可以获取图标、背景等 View 做动画 |

#### Gradle 依赖

```groovy
implementation "androidx.core:core-splashscreen:1.0.1"
```

#### 代码示例

**基础用法：**

```xml
<!-- 1. res/values/themes.xml 中定义启动主题 -->
<style name="Theme.App.Starting" parent="Theme.SplashScreen">
    <!-- 启动画面背景色 -->
    <item name="windowSplashScreenBackground">@color/splash_background</item>
    <!-- 启动画面中央图标 -->
    <item name="windowSplashScreenAnimatedIcon">@drawable/ic_splash</item>
    <!-- 图标背景色（圆形背景） -->
    <item name="windowSplashScreenIconBackgroundColor">@color/splash_icon_bg</item>
    <!-- 启动画面结束后切换到主主题 -->
    <item name="postSplashScreenTheme">@style/Theme.App</item>
</style>
```

```xml
<!-- 2. AndroidManifest.xml 中使用启动主题 -->
<activity
    android:name=".MainActivity"
    android:theme="@style/Theme.App.Starting"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

```java
// 3. Activity 中控制
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // 必须在 super.onCreate() 之前调用
        SplashScreen splashScreen = installSplashScreen();

        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 方式一：等数据准备好再关闭启动画面
        splashScreen.setKeepOnScreenCondition(() -> !viewModel.isReady());
    }
}
```

**自定义退出动画：**

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        SplashScreen splashScreen = installSplashScreen();

        // 设置自定义退出动画
        splashScreen.setOnExitAnimationListener(splashScreenView -> {
            // 创建退出动画
            ObjectAnimator fadeOut = ObjectAnimator.ofFloat(
                splashScreenView, View.ALPHA, 1f, 0f);
            fadeOut.setDuration(300);
            fadeOut.setInterpolator(new AccelerateInterpolator());

            // 动画结束后移除启动画面
            fadeOut.addListener(new AnimatorListenerAdapter() {
                @Override
                public void onAnimationEnd(Animator animation) {
                    splashScreenView.remove();
                }
            });

            fadeOut.start();
        });

        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

**启动画面图标动画（Android 12+）：**

```xml
<!-- res/drawable/ic_splash_animated.xml -->
<!-- 使用 AnimatedVectorDrawable 做图标动画 -->
<animated-vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:drawable="@drawable/ic_splash_static">
    <target
        android:name="icon_group"
        android:animation="@animator/splash_icon_rotation" />
</animated-vector>
```

```xml
<!-- res/animator/splash_icon_rotation.xml -->
<objectAnimator
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:propertyName="rotation"
    android:valueFrom="0"
    android:valueTo="360"
    android:duration="1000"
    android:interpolator="@android:interpolator/accelerate_decelerate" />
```

#### 最佳实践

- `installSplashScreen()` **必须**在 `super.onCreate()` 之前调用
- 启动画面图标建议用 **Vector Drawable**，大小 240x240 dp
- Android 12+ 的进入动画时长由系统控制，你只能自定义退出动画
- 启动画面期间不要做耗时操作，应该尽快显示主界面
- `windowSplashScreenAnimatedIcon` 在 Android 12+ 支持动画，低版本只显示静态图标

#### 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 启动画面闪一下就没了 | 数据加载太快 | setKeepOnScreenCondition 控制时长 |
| 退出动画没生效 | 没调 remove() | 动画结束后必须调 splashScreenView.remove() |
| 低版本没显示 | 兼容库没配置 | 检查主题和依赖 |

---

## 速查总表

| # | 库名 | 分类 | 一句话定位 | 核心原理 | Java 关键 API |
|---|------|------|-----------|---------|--------------|
| 1 | **ViewModel** | 架构 | 屏幕级状态容器，旋转不丢数据 | ViewModelStore 缓存实例，生命周期比 Activity 长 | `ViewModelProvider(owner).get(cls)` |
| 2 | **LiveData** | 架构 | 感知生命周期的可观察数据 | 关联 LifecycleOwner，活跃时才通知，销毁时自动断开 | `MutableLiveData.setValue()` / `observe(owner, observer)` |
| 3 | **Room** | 架构 | SQLite 的 ORM 封装 | 编译时注解处理生成 SQL 代码，编译时验证查询 | `@Entity` / `@Dao` / `@Query` / `Room.databaseBuilder()` |
| 4 | **DataStore** | 架构 | SP 的替代品，异步存储 | RxJava + 原子性读写，文件持久化 | `RxPreferenceDataStoreBuilder` / `updateDataAsync()` |
| 5 | **Navigation** | 架构 | 页面跳转统一管理 | 图结构管理目的地，NavController 中央调度 | `NavHostFragment` / `NavController.navigate()` / Safe Args |
| 6 | **Paging 3** | 架构 | 分页加载全家桶 | 三层架构（Source→Pager→Adapter），自动加载和缓存 | `PagingSource` / `Pager.createFlowable()` / `PagingDataAdapter` |
| 7 | **Hilt** | 架构 | 依赖注入框架 | 编译时生成 Dagger 组件，按层级自动注入 | `@HiltAndroidApp` / `@AndroidEntryPoint` / `@HiltViewModel` / `@Module` |
| 8 | **ViewBinding** | UI | XML 布局类型安全绑定 | 编译时生成绑定类，替代 findViewById | `XxxBinding.inflate()` / `binding.getRoot()` |
| 9 | **Fragment** | UI | 可插拔的 UI 模块 | 独立生命周期，FragmentManager 管理事务 | `newInstance()` / `getViewLifecycleOwner()` / Fragment Result API |
| 10 | **RecyclerView** | UI | 高效列表控件 | ViewHolder 复用 + DiffUtil 差异更新 | `ListAdapter` / `DiffUtil.ItemCallback` / `submitList()` |
| 11 | **WorkManager** | 基础 | 可靠的后台任务调度 | SQLite 持久化 + JobScheduler/AlarmManager 自适应 | `Worker` / `RxWorker` / `OneTimeWorkRequest` / `WorkManager.enqueue()` |
| 12 | **Startup** | 基础 | 启动初始化统一管理 | 共享 ContentProvider + Initializer 依赖排序 | `Initializer<T>` / `create()` / `dependencies()` |
| 13 | **SplashScreen** | 基础 | 统一启动画面 | 兼容 Android 12+ 系统行为，三阶段动画 | `installSplashScreen()` / `setKeepOnScreenCondition()` |

---

> 📖 所有内容基于 [developer.android.com/jetpack](https://developer.android.com/jetpack) 官方文档整理，如有出入以官方文档为准。
