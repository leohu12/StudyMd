# Android 速查手册

> Android 开发最常用的 API 和代码片段

---

## Activity

### 生命周期

```kotlin
class MainActivity : AppCompatActivity() {
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        // 初始化视图、数据
    }
    
    override fun onStart() {
        super.onStart()
        // 界面可见
    }
    
    override fun onResume() {
        super.onResume()
        // 可交互
    }
    
    override fun onPause() {
        super.onPause()
        // 失去焦点
    }
    
    override fun onStop() {
        super.onStop()
        // 不可见
    }
    
    override fun onDestroy() {
        super.onDestroy()
        // 清理资源
    }
    
    override fun onRestart() {
        super.onRestart()
        // 从后台返回前台
    }
}
```

### 跳转与传参

```kotlin
// 普通跳转
val intent = Intent(this, SecondActivity::class.java)
startActivity(intent)

// 带参数跳转
val intent = Intent(this, SecondActivity::class.java).apply {
    putExtra("key_string", "value")
    putExtra("key_int", 100)
    putExtra("key_boolean", true)
    putExtra("key_serializable", user)
    putExtra("key_parcelable", user)
}
startActivity(intent)

// 接收参数
val stringValue = intent.getStringExtra("key_string")
intValue = intent.getIntExtra("key_int", 0)
val user = intent.getParcelableExtra<User>("key_parcelable")

// 带返回结果的跳转
startActivityForResult(intent, REQUEST_CODE)
// 或新的 API
resultLauncher.launch(intent)

// 返回结果
val resultIntent = Intent().apply {
    putExtra("result", "success")
}
setResult(RESULT_OK, resultIntent)
finish()

// 接收返回结果
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)
    if (requestCode == REQUEST_CODE && resultCode == RESULT_OK) {
        val result = data?.getStringExtra("result")
    }
}
```

### 启动模式

```xml
<activity
    android:name=".MainActivity"
    android:launchMode="singleTop" />  <!-- standard/singleTop/singleTask/singleInstance -->
```

---

## Fragment

```kotlin
class MyFragment : Fragment(R.layout.fragment_my) {
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        // 找视图
        val textView = view.findViewById<TextView>(R.id.text_view)
        
        // 或 ViewBinding
        val binding = FragmentMyBinding.bind(view)
        binding.textView.text = "Hello"
    }
    
    companion object {
        fun newInstance(param: String) = MyFragment().apply {
            arguments = Bundle().apply {
                putString("param", param)
            }
        }
    }
}

// Activity 中使用
supportFragmentManager.beginTransaction()
    .replace(R.id.container, MyFragment.newInstance("param"))
    .addToBackStack(null)
    .commit()
```

---

## View

### 常用属性

```kotlin
view.visibility = View.VISIBLE      // VISIBLE/INVISIBLE/GONE
view.isEnabled = true
view.isClickable = true
view.alpha = 0.5f
view.rotation = 45f
view.scaleX = 1.5f
view.translationX = 100f

// 布局参数
val params = view.layoutParams as ViewGroup.MarginLayoutParams
params.setMargins(16, 16, 16, 16)
view.layoutParams = params
```

### 点击事件

```kotlin
// 方式1：Lambda
button.setOnClickListener {
    // 点击处理
}

// 方式2：函数引用
button.setOnClickListener(::onButtonClick)

// 方式3：实现接口
button.setOnClickListener(object : View.OnClickListener {
    override fun onClick(v: View?) {
        // 点击处理
    }
})

// 防重复点击
fun View.setOnSingleClickListener(interval: Long = 1000, action: (View) -> Unit) {
    var lastClickTime = 0L
    setOnClickListener {
        val currentTime = System.currentTimeMillis()
        if (currentTime - lastClickTime > interval) {
            lastClickTime = currentTime
            action(it)
        }
    }
}
```

---

## RecyclerView

```kotlin
// Adapter
class MyAdapter : RecyclerView.Adapter<MyAdapter.ViewHolder>() {
    
    private var items = listOf<Item>()
    
    fun setData(newItems: List<Item>) {
        items = newItems
        notifyDataSetChanged()
    }
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_layout, parent, false)
        return ViewHolder(view)
    }
    
    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.bind(items[position])
    }
    
    override fun getItemCount() = items.size
    
    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        fun bind(item: Item) {
            itemView.findViewById<TextView>(R.id.text).text = item.name
        }
    }
}

// 使用
recyclerView.layoutManager = LinearLayoutManager(context)
recyclerView.adapter = MyAdapter()
recyclerView.addItemDecoration(DividerItemDecoration(context, DividerItemDecoration.VERTICAL))

// 或 Grid
recyclerView.layoutManager = GridLayoutManager(context, 2)
```

### ListAdapter（带 DiffUtil）

```kotlin
class MyListAdapter : ListAdapter<Item, MyListAdapter.ViewHolder>(DiffCallback()) {
    
    class DiffCallback : DiffUtil.ItemCallback<Item>() {
        override fun areItemsTheSame(old: Item, new: Item) = old.id == new.id
        override fun areContentsTheSame(old: Item, new: Item) = old == new
    }
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
        val binding = ItemLayoutBinding.inflate(
            LayoutInflater.from(parent.context), parent, false
        )
        return ViewHolder(binding)
    }
    
    override fun onBindViewHolder(holder: ViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
    
    class ViewHolder(private val binding: ItemLayoutBinding) : 
        RecyclerView.ViewHolder(binding.root) {
        fun bind(item: Item) {
            binding.textView.text = item.name
        }
    }
}

// 提交数据
adapter.submitList(newList)
```

---

## ViewModel

```kotlin
class MyViewModel : ViewModel() {
    
    // LiveData
    private val _data = MutableLiveData<String>()
    val data: LiveData<String> = _data
    
    // StateFlow（推荐）
    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
    
    fun loadData() {
        viewModelScope.launch {
            _uiState.value = UiState.Loading
            try {
                val result = repository.fetchData()
                _uiState.value = UiState.Success(result)
            } catch (e: Exception) {
                _uiState.value = UiState.Error(e.message)
            }
        }
    }
}

// Activity 中使用
class MainActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // 观察 LiveData
        viewModel.data.observe(this) { data ->
            textView.text = data
        }
        
        // 收集 StateFlow
        lifecycleScope.launch {
            viewModel.uiState.collect { state ->
                when (state) {
                    is UiState.Loading -> showLoading()
                    is UiState.Success -> showData(state.data)
                    is UiState.Error -> showError(state.message)
                }
            }
        }
    }
}
```

---

## Room 数据库

```kotlin
// Entity
@Entity(tableName = "users")
data class User(
    @PrimaryKey val id: String,
    val name: String,
    val age: Int
)

// DAO
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAll(): List<User>
    
    @Query("SELECT * FROM users WHERE id = :userId")
    fun getById(userId: String): User?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    fun insert(user: User)
    
    @Update
    fun update(user: User)
    
    @Delete
    fun delete(user: User)
    
    @Query("DELETE FROM users")
    fun deleteAll()
    
    // 返回 Flow，自动观察变化
    @Query("SELECT * FROM users")
    fun getAllFlow(): Flow<List<User>>
}

// Database
@Database(entities = [User::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}

// 创建
val db = Room.databaseBuilder(
    applicationContext,
    AppDatabase::class.java, "my-database"
).build()

// 使用
val users = db.userDao().getAll()
```

---

## Retrofit 网络请求

```kotlin
// API 接口
interface ApiService {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: String): User
    
    @GET("users")
    suspend fun getUsers(@Query("page") page: Int): List<User>
    
    @POST("users")
    suspend fun createUser(@Body user: User): User
    
    @FormUrlEncoded
    @POST("login")
    suspend fun login(
        @Field("username") username: String,
        @Field("password") password: String
    ): LoginResponse
    
    @Multipart
    @POST("upload")
    suspend fun uploadFile(
        @Part file: MultipartBody.Part
    ): UploadResponse
}

// 创建 Retrofit
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()

val apiService = retrofit.create(ApiService::class.java)

// 使用
viewModelScope.launch {
    try {
        val user = apiService.getUser("123")
    } catch (e: Exception) {
        // 处理错误
    }
}
```

---

## 权限申请

```kotlin
// AndroidManifest.xml 中声明
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

// 运行时申请（Activity Result API）
val requestPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    if (isGranted) {
        // 权限已授予
    } else {
        // 权限被拒绝
    }
}

// 检查并申请
when {
    ContextCompat.checkSelfPermission(context, Manifest.permission.CAMERA) 
        == PackageManager.PERMISSION_GRANTED -> {
        // 已有权限
    }
    shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) -> {
        // 显示解释为什么需要权限
    }
    else -> {
        requestPermissionLauncher.launch(Manifest.permission.CAMERA)
    }
}

// 多个权限
val requestMultiplePermissions = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { permissions ->
    when {
        permissions.getOrDefault(Manifest.permission.CAMERA, false) -> {
            // 相机权限已授予
        }
        else -> {
            // 相机权限被拒绝
        }
    }
}
```

---

## SharedPreferences / DataStore

```kotlin
// SharedPreferences（旧）
val prefs = getSharedPreferences("my_prefs", Context.MODE_PRIVATE)
prefs.edit {
    putString("name", "张三")
    putInt("age", 25)
}
val name = prefs.getString("name", "")

// DataStore（新，推荐）
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

// 定义键
val NAME_KEY = stringPreferencesKey("name")
val AGE_KEY = intPreferencesKey("age")

// 写入
viewModelScope.launch {
    context.dataStore.edit { settings ->
        settings[NAME_KEY] = "张三"
        settings[AGE_KEY] = 25
    }
}

// 读取
val nameFlow: Flow<String> = context.dataStore.data
    .map { it[NAME_KEY] ?: "" }
```

---

## 通知

```kotlin
// 创建通知渠道（Android 8+）
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
    val channel = NotificationChannel(
        CHANNEL_ID,
        "频道名称",
        NotificationManager.IMPORTANCE_DEFAULT
    ).apply {
        description = "频道描述"
    }
    val notificationManager = getSystemService(NotificationManager::class.java)
    notificationManager.createNotificationChannel(channel)
}

// 创建通知
val notification = NotificationCompat.Builder(this, CHANNEL_ID)
    .setSmallIcon(R.drawable.ic_notification)
    .setContentTitle("通知标题")
    .setContentText("通知内容")
    .setPriority(NotificationCompat.PRIORITY_DEFAULT)
    .setAutoCancel(true)
    .setContentIntent(pendingIntent)  // 点击跳转
    .build()

// 发送通知
NotificationManagerCompat.from(this).notify(notificationId, notification)
```

---

## 常用工具类

### Toast

```kotlin
Toast.makeText(context, "消息", Toast.LENGTH_SHORT).show()

// 扩展函数
fun Context.toast(message: String, duration: Int = Toast.LENGTH_SHORT) {
    Toast.makeText(this, message, duration).show()
}
```

### Log

```kotlin
Log.d(TAG, "调试信息")
Log.i(TAG, "普通信息")
Log.w(TAG, "警告")
Log.e(TAG, "错误", exception)
```

### Intent 工具

```kotlin
// 打开网页
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://www.google.com"))
startActivity(intent)

// 发送邮件
val intent = Intent(Intent.ACTION_SENDTO).apply {
    data = Uri.parse("mailto:email@example.com")
    putExtra(Intent.EXTRA_SUBJECT, "主题")
    putExtra(Intent.EXTRA_TEXT, "内容")
}

// 分享文本
val intent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "分享内容")
}
startActivity(Intent.createChooser(intent, "分享到"))

// 拨打电话
val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:10086"))
```

### 屏幕相关

```kotlin
// dp 转 px
fun Context.dpToPx(dp: Float): Int {
    return (dp * resources.displayMetrics.density).toInt()
}

// px 转 dp
fun Context.pxToDp(px: Float): Float {
    return px / resources.displayMetrics.density
}

// 屏幕宽高
val displayMetrics = resources.displayMetrics
val widthPixels = displayMetrics.widthPixels
val heightPixels = displayMetrics.heightPixels
```

### 软键盘

```kotlin
// 显示软键盘
fun View.showKeyboard() {
    requestFocus()
    val imm = context.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    imm.showSoftInput(this, InputMethodManager.SHOW_IMPLICIT)
}

// 隐藏软键盘
fun View.hideKeyboard() {
    val imm = context.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    imm.hideSoftInputFromWindow(windowToken, 0)
}
```

---

## Jetpack Compose（现代 UI）

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!")
}

@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(text = "Count: $count", fontSize = 24.sp)
        Button(onClick = { count++ }) {
            Text("增加")
        }
    }
}

// 列表
@Composable
fun UserList(users: List<User>) {
    LazyColumn {
        items(users) { user ->
            UserItem(user)
        }
    }
}

@Composable
fun UserItem(user: User) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(8.dp),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Text(text = user.name, modifier = Modifier.padding(16.dp))
    }
}

// 状态管理
@Composable
fun MyScreen(viewModel: MyViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    when (uiState) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> Content((uiState as UiState.Success).data)
        is UiState.Error -> ErrorMessage((uiState as UiState.Error).message)
    }
}
```

---

> 建议配合 README.md 一起学习！
