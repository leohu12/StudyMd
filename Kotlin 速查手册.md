# Kotlin 速查手册

> 最常用的 Kotlin 语法，一行一个例子

---

## 变量与类型

```kotlin
val name = "张三"              // 只读变量（推荐）
var age = 25                  // 可变变量
val nullable: String? = null  // 可空类型

// 类型转换
val num: Int = "100".toInt()
val str: String = 100.toString()
```

---

## 字符串操作

```kotlin
val name = "张三"
val age = 25

// 字符串模板
val info = "我叫$name，今年$age岁"
val calc = "1+1=${1+1}"

// 多行字符串
val json = """
    {
        "name": "$name",
        "age": $age
    }
""".trimIndent()

// 常用方法
"hello".uppercase()      // HELLO
"hello".capitalize()     // Hello
"  hello  ".trim()        // hello
"a,b,c".split(",")       // [a, b, c]
```

---

## 条件与循环

```kotlin
// if 表达式
val max = if (a > b) a else b

// when（替代 switch）
when (x) {
    1 -> println("一")
    2, 3 -> println("二或三")
    in 4..10 -> println("四到十")
    is String -> println("是字符串")
    else -> println("其他")
}

// for 循环
for (i in 1..10) { }           // 1到10
for (i in 1 until 10) { }      // 1到9
for (i in 10 downTo 1) { }     // 10到1
for (i in 1..10 step 2) { }    // 1,3,5,7,9

// 遍历集合
for (item in list) { }
for ((index, item) in list.withIndex()) { }

list.withIndex() 是 Kotlin 中的一个标准库函数，用于在遍历列表时同时获取元素的索引和值。

基本用法
它返回一个 IndexedValue 对象的列表，每个对象包含 index 和 value 两个属性。

kotlin
val fruits = listOf("苹果", "香蕉", "橙子")

// 使用 withIndex() 遍历
for ((index, value) in fruits.withIndex()) {
    println("索引 $index: $value")
}
// 输出：
// 索引 0: 苹果
// 索引 1: 香蕉
// 索引 2: 橙子
常见使用场景
1. 配合 forEach 使用
kotlin
fruits.withIndex().forEach { (index, value) ->
    println("$index -> $value")
}
2. 使用解构声明
kotlin
val indexedList = fruits.withIndex().toList()
// indexedList: [(index=0, value=苹果), (index=1, value=香蕉), (index=2, value=橙子)]

indexedList.forEach { (i, v) ->
    println("位置 $i 是 $v")
}
3. 链式调用
kotlin
fruits.withIndex()
    .filter { (index, _) -> index % 2 == 0 }  // 只保留偶数索引
    .forEach { (_, value) -> println(value) } // 输出：苹果 橙子
与传统的 for 循环对比
kotlin
// 传统方式
for (i in fruits.indices) {
    println("索引 $i: ${fruits[i]}")
}

// 使用 withIndex()（更简洁、更函数式）
fruits.withIndex().forEach { (i, v) ->
    println("索引 $i: $v")
}
注意事项
withIndex() 是惰性的，返回一个 Iterable，不会立即创建新列表

适合在需要同时使用索引和元素值时的函数式编程风格

比传统的 for 循环更具可读性，尤其在链式操作中

总的来说，withIndex() 让 Kotlin 中的带索引遍历更加简洁优雅。

// while/do-while
while (condition) { }
do { } while (condition)
```

---

## 集合操作

```kotlin
// 创建集合
val list = listOf(1, 2, 3)           // 不可变列表
val mutableList = mutableListOf(1, 2, 3)  // 可变列表
val set = setOf(1, 2, 3)
val map = mapOf("a" to 1, "b" to 2)

// 常用操作
list.size                    // 大小
list[0]                      // 取值
list.first()                 // 第一个
list.last()                  // 最后一个
list.isEmpty()               // 是否为空
list.contains(2)             // 是否包含

// 高阶函数
list.filter { it > 1 }       // 筛选
list.map { it * 2 }          // 映射
list.find { it > 1 }         // 查找第一个
list.any { it > 1 }          // 是否有满足条件的
list.all { it > 0 }          // 是否全部满足
list.none { it < 0 }         // 是否都不满足
list.count { it > 1 }        // 计数
list.sorted()                // 排序
list.sortedDescending()      // 降序
list.distinct()              // 去重
list.take(2)                 // 取前2个
list.drop(2)                 // 跳过前2个
list.slice(1..3)             // 截取
list.groupBy { it % 2 }      // 分组
list.partition { it > 2 }    // 分成两组
list.zip(anotherList)        // 合并两个列表
list.flatten()               // 扁平化
list.flatMap { it.list }     // 映射并扁平化

// 聚合
list.sum()                   // 求和
list.sumOf { it.price }      // 按字段求和
list.average()               // 平均值
list.maxOrNull()             // 最大值
list.minOrNull()             // 最小值
list.reduce { acc, i -> acc + i }      // 累积
list.fold(0) { acc, i -> acc + i }     // 带初始值的累积
```

---

## 函数

```kotlin
// 基础函数
fun greet(name: String): String {
    return "Hello, $name"
}

// 单行函数
fun add(a: Int, b: Int) = a + b

// 默认参数
fun greet(name: String, prefix: String = "Hello") = "$prefix, $name"

// 可变参数
fun sum(vararg numbers: Int) = numbers.sum()
sum(1, 2, 3, 4, 5)

// 扩展函数
fun String.addExclamation() = this + "!"
"hello".addExclamation()  // hello!

// 中缀函数
infix fun Int.add(other: Int) = this + other
5 add 3  // 8

// 函数类型
val operation: (Int, Int) -> Int = { a, b -> a + b }
```

---

## 类与对象

```kotlin
// 普通类
class Person(val name: String, var age: Int) {
    fun sayHello() = println("Hello, I'm $name")
}

// 数据类（自动生成 toString/equals/hashCode/copy）
data class User(val id: String, val name: String)
val user2 = user1.copy(name = "New Name")

// 密封类（有限继承）
sealed class Result
class Success(val data: String) : Result()
class Error(val message: String) : Result()

// 枚举
enum class Color { RED, GREEN, BLUE }

// 单例
object Singleton {
    fun doSomething() {}
}

// 伴生对象（类的静态成员）
class MyClass {
    companion object {
        const val TAG = "MyClass"
        fun create() = MyClass()
    }
}
MyClass.TAG
MyClass.create()

// 接口
interface Clickable {
    fun onClick()
    fun showOff() = println("I'm clickable")  // 默认实现
}

// 抽象类
abstract class Animal {
    abstract fun makeSound()
    fun sleep() = println("Sleeping...")
}

// 继承
class Dog : Animal() {
    override fun makeSound() = println("Woof!")
}

// 数据类解构
data class Person(val name: String, val age: Int)
val (name, age) = person
```

---

## 空安全

```kotlin
var str: String? = null

// 安全调用
str?.length                  // null 时返回 null

// Elvis 运算符
val len = str?.length ?: 0   // null 时用默认值

// 非空断言（慎用）
val len = str!!.length       // null 时抛异常

// let 处理
str?.let { println(it) }     // 不为空时执行

// 安全类型转换
val num = str as? Int        // 转换失败返回 null
```

---

## 作用域函数

```kotlin
val person = Person("张三", 25)

// let - 非空判断 + 转换
person.let { p ->
    println(p.name)
    p.age + 1
}

// run - 对象配置 + 计算
person.run {
    println(name)
    age + 1
}

// with - 对对象执行多个操作
with(person) {
    println(name)
    println(age)
}

// apply - 对象初始化配置
val p = Person("", 0).apply {
    name = "李四"
    age = 30
}

// also - 副作用操作
person.also {
    println("创建用户: ${it.name}")
}
```

| 函数 | 对象引用 | 返回值 | 用途 |
|------|----------|--------|------|
| let | it | Lambda 结果 | 非空判断 + 转换 |
| run | this | Lambda 结果 | 对象配置 + 计算 |
| with | this | Lambda 结果 | 对对象执行多个操作 |
| apply | this | 对象本身 | 对象初始化配置 |
| also | it | 对象本身 | 副作用操作 |

---

## 协程

```kotlin
import kotlinx.coroutines.*

// 启动协程
GlobalScope.launch { }                    // 全局作用域（不推荐）
lifecycleScope.launch { }                 // 生命周期作用域
viewModelScope.launch { }                 // ViewModel 作用域
coroutineScope.launch { }                 // 子作用域

// 异步获取结果
val deferred = async { fetchData() }
val result = deferred.await()

// 切换线程
withContext(Dispatchers.IO) { }           // IO 线程
withContext(Dispatchers.Default) { }      // 计算线程
withContext(Dispatchers.Main) { }         // 主线程

// 并发执行
val result1 = async { fetch1() }
val result2 = async { fetch2() }
val r1 = result1.await()
val r2 = result2.await()

// 超时
withTimeout(5000) { fetchData() }
withTimeoutOrNull(5000) { fetchData() }

// 取消
coroutineScope {
    val job = launch { fetchData() }
    job.cancel()
    job.join()  // 等待完成
}

// 异常处理
coroutineScope {
    try {
        fetchData()
    } catch (e: Exception) {
        // 处理异常
    }
}

// supervisorScope - 子协程失败不影响其他
supervisorScope {
    launch { mayFail1() }
    launch { mayFail2() }  // 失败不会取消第一个
}
```

---

## Flow

```kotlin
import kotlinx.coroutines.flow.*

// 创建 Flow
val flow = flow {
    emit(1)
    emit(2)
    emit(3)
}

val flow = listOf(1, 2, 3).asFlow()
val flow = flowOf(1, 2, 3)

// 中间操作
flow
    .filter { it > 1 }           // 筛选
    .map { it * 2 }              // 映射
    .take(2)                     // 取前2个
    .drop(1)                     // 跳过1个
    .distinctUntilChanged()      // 去重连续重复值
    .debounce(300)               // 防抖
    .sample(1000)                // 采样
    .catch { e -> }              // 捕获异常
    .onEach { println(it) }      // 副作用
    .onStart { }                 // 开始时的操作
    .onCompletion { }            // 完成时的操作

// 终端操作
flow.collect { }                 // 收集
flow.first()                     // 第一个
flow.toList()                    // 转为列表
flow.reduce { acc, value -> acc + value }  // 累积
flow.fold(0) { acc, value -> acc + value } // 带初始值累积
flow.count()                     // 计数

// 状态 Flow
val stateFlow = MutableStateFlow(initialValue)
val value = stateFlow.value      // 获取当前值
stateFlow.value = newValue       // 设置值
stateFlow.collect { }            // 收集变化

// 共享 Flow
val sharedFlow = MutableSharedFlow<Int>()
sharedFlow.emit(value)           // 发射
sharedFlow.collect { }           // 收集
```

---

## 委托

```kotlin
// 延迟初始化
val lazyValue: String by lazy {
    println("初始化！")
    "Hello"
}

// 可观察属性
var name: String by Delegates.observable("") { prop, old, new ->
    println("$old -> $new")
}

// 可否决属性
var age: Int by Delegates.vetoable(0) { prop, old, new ->
    new >= 0  // 只有非负数才允许赋值
}

// 属性映射
class User(map: Map<String, Any>) {
    val name: String by map
    val age: Int by map
}
```

---

## 类型检查与转换

```kotlin
// 类型检查
if (obj is String) { }
if (obj !is String) { }

// 智能转换
if (obj is String) {
    println(obj.length)  // 自动转为 String
}

// 显式转换
val num: Int = str as Int          // 不安全，失败抛异常
val num: Int? = str as? Int        // 安全，失败返回 null

// 多态处理
when (obj) {
    is String -> obj.length
    is Int -> obj.toString()
    else -> "Unknown"
}
```

---

## 常用技巧

```kotlin
// 交换变量
var a = 1
var b = 2
a = b.also { b = a }

//  try-catch 表达式
val result = try {
    parseNumber(str)
} catch (e: Exception) {
    0
}

//  if-not-null 简写
val files = File("/path").listFiles()
println(files?.size ?: "empty")

//  if-not-null-else 简写
files?.size ?: run {
    println("没有文件")
}

// 返回 when
fun describe(obj: Any): String = when (obj) {
    is String -> "String"
    is Int -> "Int"
    else -> "Unknown"
}

// 使用 apply 构建对象
val person = Person().apply {
    name = "张三"
    age = 25
}

// 使用 run 计算多个值
val (min, max) = listOf(1, 2, 3).run {
    minOrNull() to maxOrNull()
}

// 使用 with 操作同一个对象多次
with(buffer) {
    append("Hello")
    append(" ")
    append("World")
}

// 使用 also 打印调试
list.filter { it > 0 }
    .also { println("筛选后: $it") }
    .map { it * 2 }
```

---

> 建议收藏，随时查阅！
