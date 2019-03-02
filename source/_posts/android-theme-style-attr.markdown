---
title: "Android之Theme、Style、Attr"
date: 2017-11-09 19:15:01 +0800
tags:
 - UI
---



Android UI开发中经常会涉及到Theme、Style、Attr等概念，熟悉掌握这些概念能够帮助我们快速实现想要的UI效果，另外自定义View也经常需要使用到这些东西。

<!-- more -->

### 概念

* **Attr** 属性——基础单元，在Theme/Style/XML文件中作为Key使用，指定相应的value。

定义方式：

```xml
<attr name="borderWidth" format="dimen" />
```

使用方式:

```xml
<View
      android:layout_width="wrap_content"
      android:layout_height="wrap_content"
      app:borderWidth="10dp" />
```

或

```java
int attr = R.attr.borderWidth;
```



可以将多个关联属性分组管理:

```java
<declare-styleable name="MyButton">
    <attr name="buttonWidth" format="dimension" />
    <attr name="buttonHeight" format="dimension" />
    <attr name="buttonColor" format="color" />
</declare-styleable>
```

通过以下方式可以访问到一个属性数组:

```java
int[] attrs = R.styleable.MyButton;
```



* **Style** 样式集合，将多个属性放在一起，达到复用的目的。

例如：

```xml
<Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:textSize="20sp"
        android:textColor="#FF000" />
```

抽离出一些公共的属性作为Style:

```xml
<style name="myButtonStyle">
        <item name="android:layout_width">wrap_content</item>
        <item name="android:layout_height">wrap_content</item>
        <item name="android:gravity">center</item>
        <item name="android:textColor">#FF0000</item>
        <item name="android:textSize">20sp</item>
</style>
```

引用Style:

```xml
<Button
        android:id="@+id/button"
        style="@style/myButtonStyle" />
```

* **Theme** 主题，相当于一个大的Style，作用在应用的层次。其中会包含一些Window相关的属性，比如:
```xml
<item name="windowBackground">?attr/colorBackground</item>
<item name="windowClipToOutline">true</item>
<item name="windowFrame">@null</item>
<item name="windowNoTitle">false</item>
<item name="windowFullscreen">false</item>
<item name="windowOverscan">false</item>
<item name="windowIsFloating">false</item>
```

一些组件(Dialog，View等）的统一样式，比如：
```xml
<item name="dialogTheme">@style/ThemeOverlay.Material.Dialog</item>
<item name="dialogTitleDecorLayout">@layout/dialog_title_material</item>
<item name="dialogPreferredPadding">@dimen/dialog_padding_material</item>
<item name="searchViewStyle">@style/Widget.Material.SearchView</item>
<item name="searchDialogTheme">@style/Theme.Material.SearchBar</item>
<item name="numberPickerStyle">@style/Widget.Material.NumberPicker</item>
<item name="calendarViewStyle">@style/Widget.Material.CalendarView</item>
<item name="timePickerStyle">@style/Widget.Material.TimePicker</item>
<item name="timePickerDialogTheme">?attr/dialogTheme</item>
<item name="datePickerStyle">@style/Widget.Material.DatePicker</item>
<item name="datePickerDialogTheme">?attr/dialogTheme</item>
```

主题相当于应用的一套皮肤，这套皮肤制定了各个组件的显示风格，使之具有统一性。我们熟知的有`Theme.Holo`,`Theme.Material`等等。


### Style、Theme作用在View上的流程

**问题：**既然使用Style、Theme都可以给View一个样式，那么他们是怎样作用在View上的呢？他们两个的优先级又是怎么样的。

这里说一下优先级，日常的开发中应该都能够得出一个经验：**layout布局文件中属性 > style样式 > Theme主题**

拿一个Button举例，如果在布局文件中给Button设置了`android:background="XXX"` 或者抽离到Style中再应用，那么Button就显示了我们指定的背景。 如果没有设置背景属性，Button仍然是有一个背景的。这个默认背景就是应用到了Theme中的样式，并且对于不同的主题有不同的样式~

然后重点来说一下样式是怎样作用到View上的，对这个过程进行深入的理解。同样拿一个Android的View来举例：`TextView`，看看它是怎么应用样式的。



[@TextView] 构造方法:

```java
public TextView(Context context) {
        this(context, null);
}

public TextView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, com.android.internal.R.attr.textViewStyle);
}

public TextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
}

@SuppressWarnings("deprecation")
public TextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
}
```

我们继承Android的View来自定义View时，通过会被要求继承四个的构造方法中的一个。对于XML中布局的View，被调用2个参数的构造方法来new一个实例，其中attrs就是布局的属性集，其中包含了这个View的所有样式。

TextView所有的构造函数最终都指向最长参数的构造函数：

```java
public TextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        final Resources.Theme theme = context.getTheme();
        TypedArray a = theme.obtainStyledAttributes(attrs,
                com.android.internal.R.styleable.TextViewAppearance, defStyleAttr, defStyleRes);
        TypedArray appearance = null;
        int ap = a.getResourceId(
                com.android.internal.R.styleable.TextViewAppearance_textAppearance, -1);
        a.recycle();
        if (ap != -1) {
            appearance = theme.obtainStyledAttributes(
                    ap, com.android.internal.R.styleable.TextAppearance);
        }
        if (appearance != null) {
            int n = appearance.getIndexCount();
            for (int i = 0; i < n; i++) {
                int attr = appearance.getIndex(i);

                switch (attr) {
                case com.android.internal.R.styleable.TextAppearance_textColorHighlight:
                    textColorHighlight = appearance.getColor(attr, textColorHighlight);
                    break;

                case com.android.internal.R.styleable.TextAppearance_textColor:
                    textColor = appearance.getColorStateList(attr);
                    break;
                //省略大量Case
            }

            appearance.recycle();
        }

        a = theme.obtainStyledAttributes(attrs, com.android.internal.R.styleable.TextView, defStyleAttr, defStyleRes);

        int n = a.getIndexCount();
        for (int i = 0; i < n; i++) {
            int attr = a.getIndex(i);

            switch (attr) {
            case com.android.internal.R.styleable.TextView_editable:
                editable = a.getBoolean(attr, editable);
                break;

            case com.android.internal.R.styleable.TextView_inputMethod:
                inputMethod = a.getText(attr);
                break;
            //省略大量Case
        }
}
```



其中最核心的一个方法是`context.obtainStyledAttributes(AttributeSet, int[] attrs, defStyleAttr, defStyleRes)`。

**AttributeSet** : layout文件中解析出来的属性对象集合，包含我们的样式。

**attrs**: 前面讲到的一组相关联的属性集合。`com.android.internal.R.styleable.TextView` 可在AOSP中查看具体有哪些属性，这里列出一部分：

```xml
 <declare-styleable name="TextView">
        <!-- Determines the minimum type that getText() will return.
             The default is "normal".
             Note that EditText and LogTextBox always return Editable,
             even if you specify something less powerful here. -->
        <attr name="bufferType">
            <!-- Can return any CharSequence, possibly a
             Spanned one if the source text was Spanned. -->
            <enum name="normal" value="0" />
            <!-- Can only return Spannable. -->
            <enum name="spannable" value="1" />
            <!-- Can only return Spannable and Editable. -->
            <enum name="editable" value="2" />
        </attr>
        <!-- Text to display. -->
        <attr name="text" format="string" localization="suggested" />
        <!-- Hint text to display when the text is empty. -->
        <attr name="hint" format="string" />
        <!-- Text color. -->
        <attr name="textColor" />
        <!-- Color of the text selection highlight. -->
        <attr name="textColorHighlight" />
        <!-- Color of the hint text. -->
        <attr name="textColorHint" />
        <!-- Base text color, typeface, size, and style. -->
        <attr name="textAppearance" />
        <!-- Size of the text. Recommended dimension type for text is "sp" for scaled-pixels (example: 15sp). -->
        <attr name="textSize" />
        <!-- Sets the horizontal scaling factor for the text. -->
        <attr name="textScaleX" format="float" />
   
   <!--省略不少-->
```

**defStyleAttr** ： 一个指定的属性资源。在这里为 `com.android.internal.R.attr.textViewStyle`（2个参数的构造方法传进来的）。可以在**Theme**中找到此属性对应的值，对应了一个Style.

```xml
<item name="textViewStyle">@style/Widget.Material.Light.TextView</item>
```

**defStyleRes**:  Style资源，也是一组样式。



以上，`context.obtainStyledAttributes `  获取View样式的过程为：

1. 从AttributeSet样式集合中寻找`int[] attrs`指定的几个属性对应的值。例如：xml中指定了`android:textColor="#ff0000"`， attrs属性组中定义有`textColor`这个属性，则提取出来。

2. 如果AttributeSet中没有要提取的样式（比如，以上没有指定textColor样式），则根据**defStyleAttr**来从指定的**Theme**中寻找样式。比如：Material主题中指定了:

```xml
<item name="textViewStyle">@style/Widget.Material.Light.TextView</item>
```

则进一步去`@style/Widget.Material.Light.TextView` 中寻找想要的样式。

3. 如果主题中仍然找不到要提取的样式。 则去**defStyleRes**(我们指定的Style样式中)寻找。



### 另外

经过上面的分析，已经可以知道Theme是怎样应用默认样式到View上的了，因此我们就可以修改这种默认样式来定制我们自己的主题。比如我们想让默认的Button控件字体为30sp。

```xml
<style name="CustomTheme" parent="@android:style/Theme.Material">
    <item name="android:buttonStyle">@style/CustomButtonStyle</item>
</style>
    
<style name="CustomButtonStyle" parent="@android:style/Widget.Button">
    <item name="android:textSize">30sp</item>
</style>
```

首先，自定义**CustomTheme**继承Andorid的Theme，复写**buttonStyle**指向我们自定义的样式。

其次，定义我们自己的Button样式，可以继承原来的样式，复写textSize属性，来修改默认的Button字体大小。



### 再另外

前面提到**Theme**中会有一些Window的样式，我们可以复写来实现一些window的效果.比如

```xml
<style name="CustomTheme" parent="@android:style/Theme.Material">
   <item name="android:windowFullscreen">true</item> <!--全屏-->
   <item name="android:statusBarColor">#FF0000</item> <!--修改状态栏的颜色-->
   <item name="windowNoTitle">false</item> <!--无标题-->
</style>
```

这些属性在PhoneWindow的generateLayout方法中被解析和应用。



### 相关资料

* Android源码分析
* [Android沉浸状态栏的实现](http://www.jianshu.com/p/d147608dc27b)