---
layout: post
title: Android View - Flutter Widget
date: 2020-08-13 11:15:07
updated: 2020-08-13 11:15:07
tags:
  - flutter
  - widget
categories: Flutter
---

主要列举 Android 常用的 View 对应 Flutter 的 Widget

## TextView - Text

```xml
<TextView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Hello World!"
    android:textColor="@android:colorholo_red_dark"
    android:textSize="18sp" />
```

```dart
Text(
  'Hello World!',
  style: TextStyle( // 文本样式
    color: Colors.red,
    fontSize: 18,
  ),
)
```

### 富文本

在 Android 实现，比如部分文字是可点击的、不同颜色的，是通过 Spannable 实现，而在 Flutter 通过组合 TextSpan Widget 并包裹在 Text.rich 里面

```dart
Text.rich(TextSpan())
```

### 宽高限制

在 Android 的 View 中有个通用属性 maxWidth，将它设置给 TextView 可以控制其最大宽度。而在 Flutter 的 Text 中并没有该属性，而是通过组合 ConstrainedBox 控制

```dart
ConstrainedBox(
  constraints: BoxConstraints( // 对子的约束
    maxWidth: 50,
  ),
  child: Text('123456789'),
)
```

## Button - RaisedButton

```xml
<Button
    android:text="Click Me!"
    android:onClick="xxx"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"/>
```

```dart
RaisedButton(
  onPressed: () {}, // 点击回调
  child: Text('Click Me!'),
)
```

RaisedButton 并没有提供直接的 text 属性，而是通过包裹一个 child 去显示按钮上的文字，而且该 child 可以是任何 Widget 而不必须是 Text

> 从而也可看出 Flutter 的各个 Widget 都比较轻量，各自担起各自的职责，而不是像 Android View 一样拥有各种通用属性导致 View 臃肿

### 普通文本如何点击

在 Android 中，TextView 继承自 View，而 View 都可以设置点击事件。而在 Flutter 中，Text 是没有提供像 RaisedButton 的 onPressed 回调的，这样说明各个 Widget 只负责各自的功能，那 Text 如何实现点击呢

```dart
GestureDetector(
  onTap: () {}, // 点击回调
  child: Text('this is Text, try click me'),
),
```

## EditText - TextField

```xml
<EditText
    android:hint="Your Password"
    android:inputType="textPassword"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"/>
```

```dart
TextField(
  obscureText: true, // 输入的内容变**
  decoration: InputDecoration(
    labelText: 'Your Password',
  ),
  controller: _controller, // 通过控制器控制
  onChanged: (value) {}, // 文本变化
)
```

> 这里有必要介绍下 controller，controller 这个概念在 Flutter 的很多 Widget 都会出现，在 TextField 旨在控制输入框的文本、光标等。

## ImageView - Image

```xml
<ImageView
    android:src="@mipmap/ic_launcher"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"/>
```

```dart
Image(image: AssetImage('images/ic_launcher.png'))
```

### ImageProvider 是什么

上面 Flutter Image 例子中 image 属性实际是一个 ImageProvider 对象，主要是为 Image 提供图片，常用的有

- AssetImage
- FileImage
- NetworkImage

> ImageProvider 内部还实现了图片的缓存

### 控制长宽比例

利用 AspectRatio

```dart
AspectRatio(
  aspectRatio: 3 / 2,
  child: Image(
      image: NetworkImage(
          'https://flutter.github.io/assets-for-api-doassets/widgets/owl-2.jpg')),
)
```

### 圆形图片 圆角图片

1、CircleAvatar

```dart
CircleAvatar(
  backgroundImage: AssetImage('images/ic_launcher.png'),
)
```

2、Material

```dart
Material(
  child: Image.asset('images/ic_launcher.png'),
  borderRadius: BorderRadius.circular(20.0),
  shadowColor: Colors.grey,
  elevation: 5.0,
)
```

3、BoxDecoration

```dart
Container(
  decoration: BoxDecoration(
    borderRadius: BorderRadius.circular(20.0),
  ),
  child: Image.asset('images/ic_launcher.png'),
)
```

## SeekBar - Slider

```xml
<SeekBar
    android:progress="50"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>
```

```dart
Slider(
  value: _seekBarValue,
  divisions: 5, // 分块
  label: 'yep', // 气泡文案
  onChanged: (val) {}, // 0-1值监听
)
```

## LinearLayout - Row / Column

```xml
<LinearLayout
    android:orientation="horizontal"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <View
        android:background="@android:color/holo_red_dark"
        android:layout_width="0dp"
        android:layout_weight="2"
        android:layout_height="50dp"/>

    <Space
        android:layout_weight="1"
        android:layout_width="0dp"
        android:layout_height="0dp"/>

    <View
        android:background="@android:color/holo_blue_dark"
        android:layout_width="0dp"
        android:layout_weight="1"
        android:layout_height="50dp"/>
</LinearLayout>
```

```dart
Row(children: [
  Expanded(
    flex: 2, // 占比
    child: Container(
      color: Colors.red,
      height: 50,
    ),
  ),
  Spacer(), // 占空间
  Expanded(
    child: Container(
      color: Colors.blue,
      height: 50,
    ),
  ),
])
```

> 在 Flutter 中，线性布局以 Column 和 Row 实现，其中 LinearLayout 的 weight 在 Flutter 以 Expanded 代替，Expanded 继承 Flexible。Spacer 也是个 Expanded，只是 child 是个空 Widget

## FrameLayout - Stack

用 Stack 实现堆叠的视图

```dart
Stack(
  children: <Widget>[
    Padding(
      padding: const EdgeInsets.all(8.0),
      child: Container(
        width: 100,
        height: 100,
        color: Colors.red,
      ),
    ),
    Container(
      width: 90,
      height: 90,
      color: Colors.green,
    ),
    Container(
      width: 80,
      height: 80,
      color: Colors.blue,
    ),
  ],
)
```

### margin 属性去哪了

Android Layout 中有个 LayoutParams 对象，一般都是 MarginLayoutParams，子 View 可以使用 margin 属性去实现一些间距

> 而在 Flutter 中，则是通过嵌套 Padding、Container 等 Widget
>
> 或是在 Row 中，两个 Widget 直接加一个 SizeBox(width: 10)来实现

## RelativeLayout - 组合

在 Flutter 中，是没有 RelativeLayout 以及其各种 below、above 等属性。而是通过组合各种上面所说的 Widget 去实现（Container、Padding、Row、Column、Stack 等等）

```dart
Row(
  children: [
    Expanded(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Container(
            padding: const EdgeInsets.only(bottom: 8),
            child: Text(
              'Oeschinen Lake Campground',
              style: TextStyle(
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
          Text(
            'Kandersteg, Switzerland',
            style: TextStyle(
              color: Colors.grey[500],
            ),
          ),
        ],
      ),
    ),
    Icon(
      Icons.star,
      color: Colors.red[500],
    ),
    Text('41'),
  ],
)
```
