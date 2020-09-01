---
layout: post
title: Android View - Flutter Widget
date: 2020-08-13 11:15:07
updated: 2020-08-13 11:15:07
tags:
  - widget
  - different
categories: Flutter
---

主要列举Android常用的View对应Flutter的Widget

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

在Android实现，比如部分文字是可点击的、不同颜色的，是通过Spannable实现，而在Flutter通过组合TextSpan Widget并包裹在Text.rich里面

```dart
Text.rich(TextSpan())
```

### 宽高限制

在Android的View中有个通用属性maxWidth，将它设置给TextView可以控制其最大宽度。而在Flutter的Text中并没有该属性，而是通过组合ConstrainedBox控制

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

RaisedButton并没有提供直接的text属性，而是通过包裹一个child去显示按钮上的文字，而且该child可以是任何Widget而不必须是Text

> 从而也可看出Flutter的各个Widget都比较轻量，各自担起各自的职责，而不是像Android View一样拥有各种通用属性导致View臃肿

### 普通文本如何点击

在Android中，TextView继承自View，而View都可以设置点击事件。而在Flutter中，Text是没有提供像RaisedButton的onPressed回调的，这样说明各个Widget只负责各自的功能，那Text如何实现点击呢

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

> 这里有必要介绍下controller，controller这个概念在Flutter的很多Widget都会出现，在TextField旨在控制输入框的文本、光标等。

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

### ImageProvider是什么

上面Flutter Image例子中image属性实际是一个ImageProvider对象，主要是为Image提供图片，常用的有

- AssetImage
- FileImage
- NetworkImage

> ImageProvider内部还实现了图片的缓存

### 控制长宽比例

利用AspectRatio

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

> 在Flutter中，线性布局以Column和Row实现，其中LinearLayout的weight在Flutter以Expanded代替，Expanded继承Flexible。Spacer也是个Expanded，只是child是个空Widget

## FrameLayout - Stack

用Stack实现堆叠的视图

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

### margin属性去哪了

Android Layout中有个LayoutParams对象，一般都是MarginLayoutParams，子View可以使用margin属性去实现一些间距

> 而在Flutter中，则是通过嵌套Padding、Container等Widget
> 
> 或是在Row中，两个Widget直接加一个SizeBox(width: 10)来实现

## RelativeLayout - 组合

在Flutter中，是没有RelativeLayout以及其各种below、above等属性。而是通过组合各种上面所说的Widget去实现（Container、Padding、Row、Column、Stack等等）

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
