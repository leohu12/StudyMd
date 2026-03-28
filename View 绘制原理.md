# 🎨 Android View 绘制原理完全解析

> 用盖房子的比喻，彻底搞懂 View 的 Measure → Layout → Draw 三大流程

---

## 📚 目录

- [一、绘制三步曲](#一绘制三步曲)
- [二、递归流程详解](#二递归流程详解)
- [三、自定义 View](#三自定义-view)
- [四、自定义 ViewGroup](#四自定义-viewgroup)
- [五、性能优化建议](#五性能优化建议)

---

## 一、绘制三步曲

Android 的 View 绘制就像**装修一间毛坯房**，核心就是三步：

| 步骤 | 对应方法 | 大白话 | 类比 |
|------|----------|--------|------|
| **Measure** | `onMeasure()` | 系统问每个控件：“你要占多大地方？” | 量尺寸 |
| **Layout** | `onLayout()` | 父容器告诉子控件：“你放在哪个位置。” | 摆位置 |
| **Draw** | `onDraw()` | 每个控件在自己的区域里把自己画出来。 | 刷油漆 |

### 1.1 Measure（量尺寸）

系统从上到下递归地问每个控件需要多大空间：
根容器问子控件："你需要多大？"
├── 子控件如果是 ViewGroup，继续问它的孩子
└── 子控件如果是叶子 View，测量自己后回答


### 1.2 Layout（摆位置）

尺寸确定后，父容器为每个子控件分配坐标：
父容器告诉子控件："你放在 (0,0) 到 (500,200) 这个区域里"
├── 子控件如果是 ViewGroup，继续为它的孩子分配位置
└── 子控件如果是叶子 View，记录自己的坐标

text

### 1.3 Draw（刷油漆）

位置定好后，每个控件画自己的内容：
ViewGroup 画背景 → 画内容 → 让子 View 画自己
叶子 View 直接画自己的内容

text

---

## 二、递归流程详解

整个 View 树是一棵**树形结构**：
┌─────────────┐
│ DecorView │ ← 根（ViewGroup）
│ (根容器) │
└──────┬──────┘
│
┌───────────────┼───────────────┐
▼ ▼ ▼
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ LinearLayout│ │ FrameLayout│ │ TextView │
│ (ViewGroup) │ │ (ViewGroup) │ │ (叶子View) │
└──────┬──────┘ └─────────────┘ └─────────────┘
│
┌──────┴──────┐
▼ ▼
┌─────────┐ ┌─────────┐
│ Button │ │ TextView│
│(叶子View)│ │(叶子View)│
└─────────┘ └─────────┘

text

### 2.1 Measure 递归过程

```java
// 伪代码：每个 ViewGroup 的 onMeasure 都做这件事
protected void onMeasure(int widthSpec, int heightSpec) {
    for (View child : children) {
        // 测量孩子（如果孩子是 ViewGroup，它内部会继续测量它的孩子）
        measureChild(child, widthSpec, heightSpec);
    }
    // 根据所有孩子的尺寸，确定自己的尺寸
    setMeasuredDimension(finalWidth, finalHeight);
}
2.2 Layout 递归过程
java
// 伪代码：每个 ViewGroup 的 onLayout 都做这件事
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    for (View child : children) {
        // 计算孩子的位置
        int left = ...;
        int top = ...;
        int right = left + child.getMeasuredWidth();
        int bottom = top + child.getMeasuredHeight();
        // 调用孩子的 layout（如果孩子是 ViewGroup，它内部会继续为它的孩子安排位置）
        child.layout(left, top, right, bottom);
    }
}
2.3 Draw 递归过程
java
// 伪代码：ViewGroup 的 draw 方法
public void draw(Canvas canvas) {
    drawBackground(canvas);      // 画背景
    onDraw(canvas);              // 画自己的内容
    dispatchDraw(canvas);        // 让子 View 画自己
}

// dispatchDraw 遍历所有子 View
protected void dispatchDraw(Canvas canvas) {
    for (View child : children) {
        child.draw(canvas);      // 子 View 如果是 ViewGroup，继续递归
    }
}
三、自定义 View
自定义 View 是做一个独立的控件（如圆形按钮、进度条），不包含子 View。

3.1 核心方法
方法	职责	必须重写？
onMeasure()	告诉系统你要多大	✅ 必须
onDraw()	画出你的样子	✅ 必须
onSizeChanged()	尺寸变化时的回调	❌ 可选
3.2 完整示例：一个红色的圆形 View
java
public class CircleView extends View {
    
    private Paint paint;
    
    public CircleView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }
    
    private void init() {
        paint = new Paint();
        paint.setColor(Color.RED);
        paint.setStyle(Paint.Style.FILL);
        paint.setAntiAlias(true);
    }
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // 1. 解析父容器给的尺寸要求
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);
        
        // 2. 计算默认尺寸（100dp）
        int defaultSize = dpToPx(100);
        
        // 3. 确定最终尺寸
        int finalWidth = (widthMode == MeasureSpec.EXACTLY) ? widthSize : defaultSize;
        int finalHeight = (heightMode == MeasureSpec.EXACTLY) ? heightSize : defaultSize;
        
        // 4. 保持正方形
        int size = Math.min(finalWidth, finalHeight);
        setMeasuredDimension(size, size);
    }
    
    @Override
    protected void onDraw(Canvas canvas) {
        int centerX = getWidth() / 2;
        int centerY = getHeight() / 2;
        int radius = Math.min(getWidth(), getHeight()) / 2;
        canvas.drawCircle(centerX, centerY, radius, paint);
    }
    
    private int dpToPx(int dp) {
        return (int) (dp * getResources().getDisplayMetrics().density);
    }
}
3.3 使用方式
xml
<com.yourpackage.CircleView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:padding="16dp" />
3.4 关键注意事项
⚠️ 必须调用 setMeasuredDimension()，否则会崩溃

⚠️ 处理 wrap_content 时需要自己决定默认尺寸

⚠️ onDraw() 里不要做耗时操作，保持流畅

四、自定义 ViewGroup
自定义 ViewGroup 是做一个可以装子 View 的容器（如流式布局、自定义九宫格）。

4.1 核心方法
方法	职责	必须重写？
onMeasure()	先测量所有子 View，再确定自己多大	✅ 必须
onLayout()	给每个子 View 分配位置	✅ 必须
4.2 完整示例：一个垂直排列的简单布局
java
public class SimpleVerticalLayout extends ViewGroup {
    
    public SimpleVerticalLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }
    
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int childCount = getChildCount();
        int totalHeight = 0;   // 累计所有子 View 的高度
        int maxWidth = 0;       // 最宽的子 View 宽度
        
        // 1. 测量所有子 View
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            measureChild(child, widthMeasureSpec, heightMeasureSpec);
            
            int childWidth = child.getMeasuredWidth();
            int childHeight = child.getMeasuredHeight();
            
            totalHeight += childHeight;
            maxWidth = Math.max(maxWidth, childWidth);
        }
        
        // 2. 加上 padding 的影响
        totalHeight += getPaddingTop() + getPaddingBottom();
        maxWidth += getPaddingLeft() + getPaddingRight();
        
        // 3. 确定自己的最终尺寸
        int width = resolveSize(maxWidth, widthMeasureSpec);
        int height = resolveSize(totalHeight, heightMeasureSpec);
        setMeasuredDimension(width, height);
    }
    
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        int childCount = getChildCount();
        int currentTop = getPaddingTop();
        
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            int childWidth = child.getMeasuredWidth();
            int childHeight = child.getMeasuredHeight();
            
            // 计算位置（左对齐）
            int left = getPaddingLeft();
            int right = left + childWidth;
            int top = currentTop;
            int bottom = top + childHeight;
            
            // 分配位置
            child.layout(left, top, right, bottom);
            
            // 更新下一个子 View 的顶部位置
            currentTop = bottom;
        }
    }
}
4.3 使用方式
xml
<com.yourpackage.SimpleVerticalLayout
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="16dp">
    
    <TextView
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="第一个子 View" />
    
    <Button
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="第二个子 View" />
        
</com.yourpackage.SimpleVerticalLayout>
4.4 关键注意事项
⚠️ 必须为每个子 View 调用 layout()，否则子 View 不可见

⚠️ 记得处理 padding 和 margin

⚠️ 使用 resolveSize() 方法处理父容器的限制

五、性能优化建议
5.1 避免过度嵌套
text
❌ 不好的做法：
LinearLayout
    └── LinearLayout
        └── LinearLayout
            └── TextView

✅ 推荐做法：
ConstraintLayout
    └── TextView（使用约束定位）
5.2 减少 onDraw 中的对象创建
java
// ❌ 错误：每次绘制都创建新对象
@Override
protected void onDraw(Canvas canvas) {
    Paint paint = new Paint();  // 不要在这里创建
    canvas.drawCircle(x, y, r, paint);
}

// ✅ 正确：在构造方法中创建，复用对象
private Paint paint;

public CircleView(...) {
    paint = new Paint();
}

@Override
protected void onDraw(Canvas canvas) {
    canvas.drawCircle(x, y, r, paint);
}
5.3 使用 ViewStub 延迟加载
xml
<ViewStub
    android:id="@+id/stub"
    android:layout="@layout/complex_layout"
    android:layout_width="match_parent"
    android:layout_height="wrap_content" />
java
// 需要时再加载
ViewStub stub = findViewById(R.id.stub);
stub.inflate();
📖 总结
概念	核心职责	关键方法
View	独立控件，自己画自己	onMeasure()、onDraw()
ViewGroup	容器，管理子 View	onMeasure()、onLayout()
Measure	测量尺寸	measureChild()、setMeasuredDimension()
Layout	分配位置	child.layout()
Draw	绘制内容	canvas.drawXXX()
一句话总结：系统从上到下递归地询问每个控件要多大、放哪儿，最后每个控件在自己的区域里把自己画出来。
