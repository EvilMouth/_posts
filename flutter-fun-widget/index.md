---
layout: post
title: Fun Flutter Widget
date: 2020-08-14 11:18:09
updated: 2020-08-14 11:18:09
tags:
  - flutter
  - widget
categories: Flutter
---

本文介绍日常开发中比较实用的 Flutter Widget

## Placeholder 占位图

常用在异步加载请求时先使用 Placeholder 代替显示，请求结束后替换

```dart
const Placeholder({
  Key key,
  this.color = const Color(0xFF455A64), // BlueGrey 700
  this.strokeWidth = 2.0,
  this.fallbackWidth = 400.0,
  this.fallbackHeight = 400.0,
})
```

## Form 表单

例如登陆页面，利用 Form Widget 将用户名和密码两个输入框包起来，通过 Key 进行统一校验，还自带返回按键监听

```dart
const Form({
  Key key,
  @required this.child,
  this.autovalidate = false, // 自动校验
  this.onWillPop, // 返回按键监听
  this.onChanged, // 表单内容变化
})
```

> 在 Form 里使用的输入框目前只提供 TextFormField 和 DropdownButtonFormField，两者都继承 FormField，可自行扩展

```dart
Form(
  key: _formKey, // 通过key拿到FormState
  child: Column(
    crossAxisAlignment: CrossAxisAlignment.start,
    children: <Widget>[
      TextFormField(
        decoration: const InputDecoration(
          hintText: 'Enter your email',
        ),
        validator: (value) {
          if (value.isEmpty) {
            return 'Please enter your email';
          }
          return null; // 表示无错误
        },
        onSaved: (value) {
          _formEmail = value;
        },
      ),
      //TODO 密码输入框同理
      Padding(
        padding: const EdgeInsets.symmetric(vertical: 16.0),
        child: RaisedButton(
          onPressed: () {
            // Validate will return true if the form ivalid, or false if
            // the form is invalid.
            if (_formKey.currentState.validate()) {
              // Process data.
              _formKey.currentState.save(); // 触发FormField#onSaved回调
            }
          },
          child: Text('Submit'),
        ),
      ),
    ],
  ),
)
```

### FormField 为何要被包在 Form 里面

Form 是个 StatefuleWidget，state 是 FormState，FormState 提供了一些操作 FormField 的方法，比如 validate、save。如何获取呢，原因在 FormField 的 build()方法内向上获取到 Form，并将自己注册给 Form

```dart
@override
Widget build(BuildContext context) {
  Form.of(context)?._register(this); // 将自己注册给Form
  return widget.builder(this);
}
```

> Form.of 是什么，这就要讲到 Flutter 的 InheritedWidget，InheritedWidget 是一个方便子 Widget 获取父 Widget 提供的属性的 Widget。Theme.of、Local.of 都是基于 InheritedWidget

## FutureBuilder 支持异步加载的 Widget

一般如何实现异步加载动态更新布局，自然是通过 setState 改变状态。FutureBuilder 就是这么一个封装的 Widget，应用在比较单一、一次性的场景

```dart
const FutureBuilder({
  Key key,
  this.future, // 异步任务
  this.initialData, // 初始数据，不是必要
  @required this.builder, // Widget构建
})
```

```dart
FutureBuilder<String>(
  future: Future.delayed(
      Duration(seconds: 2), () => 'this is data from net'),
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.done) {
      if (snapshot.hasError) {
        return Text("Error: ${snapshot.error}"); // 加载失败
      } else {
        return Text("Contents: ${snapshot.data}"); // 加载成功
      }
    } else {
      return CircularProgressIndicator(); // 加载中
    }
  },
)
```

> builder 会回调多次，原因就是 snapshot 是一个状态，会改变多次

## WillPopScope 监听 Android 返回按钮

Android 才需要的返回按键监听，例如应用在双击返回键退出应用等操作

> 上面介绍的 Form 也有一个 onWillPop 属性，其实就是包了一层 WillPopScope

```dart
const WillPopScope({
  Key key,
  @required this.child,
  @required this.onWillPop, // Future<bool> Function()
})
```

下面的例子通过 WillPopScope 包裹 AlertDialog 来实现一个无法通过返回按键取消的弹窗，只需要 onWillPop 返回 flase 即表示屏蔽返回

```dart
showDialog(
  context: context,
  barrierDismissible: false,
  builder: (context) {
    return WillPopScope(
      onWillPop: () async => false,
      child: AlertDialog(
        title: Text('弹窗标题'),
        content: Text('只能通过关闭按钮关闭'),
        actions: <Widget>[
          FlatButton(
            onPressed: () {
              Navigator.of(context).pop();
            },
            child: Text('关闭'),
          ),
        ],
      ),
    );
  },
)
```

## Overlay 实现悬浮按钮

> Flutter 的路由实际就是用 Overlay 实现
>
> 可以理解为 Flutter 的世界只有一个 Activity，不同的 Page 其实就是 Fragment 堆叠
>
> 所以在 Flutter 实现全局悬浮按钮简直不要太简单

> Overlay 跟着 Navigator 一起被创建，所以通过直接 Overlay.of 取得 OverlayState

下面的例子展示如何悬浮一个全局按钮

```dart
@override
void initState() {
  super.initState();
  Future.delayed(Duration(seconds: 0)).then((value) {
    Overlay.of(context).insert(floatingView); // 插入
  });
}

final floatingView = OverlayEntry(builder: (ctx) { // OverlayEntry
  return FloatingView();
});

// 悬浮按钮
class FloatingView extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    double size = 50;
    double left = MediaQuery.of(context).size.width / 4 - size / 2;
    double top = MediaQuery.of(context).size.height / 1.5 - size / 2;
    return ChangeNotifierProvider(
      create: (context) => _PositionProvider(size, left, top),
      child: _PositionObserver(
        child: _PositionConsumer(),
      ),
    );
  }
}
```

> 如果需要移除，floatingView.remove()
>
> 利用 insert 和 remove 即可实现 Flutter 层面的 Toast

## AbsorbPointer 快速禁止屏幕点击

通过 AbsorbPointer 包裹住 RaisedButton，即使 RaisedButton 声明了点击事件，但是 AbsorbPointer 也声明了 absorbing: true，所有 RaisedButton 是点不了的

```dart
AbsorbPointer(
  absorbing: true, // 是否吸收触摸事件
  child: RaisedButton(onPressed: () {}),
)
```

> Navigator 其实也包裹了一层 AbsorbPointer，在路由跳转过程，会吸收所有屏幕触摸事件，让跳转完成

## Dismissible 列表 Item 滑动移除

用来实现滑动删除场景

```dart
ListView.builder(
  itemCount: _funItems.length,
  itemBuilder: (context, i) {
    return Dismissible(
      key: ValueKey(_funItems[i]),
      child: Container(
        color: Colors.green,
        child: ListTile(
          title: Text(_funItems[i]),
        ),
      ),
      background: Container( // 右滑背景
        color: Colors.blue,
      ),
      secondaryBackground: Container( // 左滑背景
        color: Colors.red,
      ),
      onDismissed: (decoration) {
        setState(() {
          _funItems.removeAt(i);
        });
      },
    );
  },
)
```

## Hero 图片过渡效果

在 Android 21 以上，可以通过给 View 设置一个 flag 来让页面跳转时两个 View 直接有个过渡效果，通常用于浏览大图。在 Flutter 也是类似

```dart
const Hero({
  Key key,
  @required this.tag, // String 标记
  this.createRectTween, // 过渡动画，默认为MaterialRectArcTween
  this.flightShuttleBuilder, // 允许过渡中不使用child
  this.placeholderBuilder, // 占位
  this.transitionOnUserGestures = false, // iOS滑动返回时是否也过渡
  @required this.child,
})
```

```dart
Hero(
  tag: 'hero',
  child: Image.asset('images/ic_launcher.png'),
)
```

> 只需要将 Image 包裹在 Hero 中并提供一个 tag，在第二个页面同样标记，剩下的就交给 Flutter 帮我们实现过渡

## FractionallySizedBox 屏幕百分比

通常用在某些 Widget 需要撑满屏幕

```dart
const FractionallySizedBox({
  Key key,
  this.alignment = Alignment.center,
  this.widthFactor, // 0-1 1表示撑满
  this.heightFactor,
  Widget child,
})
```

## SizeBox 控制 Widget 大小

字面意思，向父 Widget 申请一定大小的空间

```dart
SizeBox(width: 10, height: 10, child: null)
```

- 充当两个 Widget 直接的 Margin
- 控制 Widget 大小

## Transform 改变 Widget 形态

与 Android 的作用一致，可以对 Widget 进行形态上的改变，比如伪 3D 旋转

```dart
Transform(
  alignment: Alignment.topRight,
  transform: Matrix4.skewY(0.3)..rotateZ(-3.1415926 / 12.0),
  child: Container(
    padding: const EdgeInsets.all(8.0),
    color: const Color(0xFFE8581C),
    child: const Text('Apartment for rent!'),
  ),
)
```

## IndexStack 快速切换 Widget

例如某个 Widget 拥有很多种状态，每个状态不同的 Icon，通过 IndexStack 先将所有 Icon 提供，通过改变 index 快速切换 Icon

```dart
IndexedStack(
  index: _indexStackIndex,
  children: <Widget>[
    Icon(Icons.star),
    Icon(Icons.account_box),
    Icon(Icons.cake),
  ],
)
```

> IndexedStack 继承 Stack，并重写了其渲染部分，只渲染 index 命中的 Icon

## Wrap 流式布局

> Wrap 基本与 Row/Column 一致，不同的一点是当 children 超出范围的话，Wrap 会自动将超出的 chilren 换行

下面的例子中第三第四个 Chip 后被 layout 在第二行

```dart
Wrap(
  spacing: 8.0, // gap between adjacent chips
  runSpacing: 4.0, // gap between lines
  children: <Widget>[
    Chip(
      avatar: CircleAvatar(
          backgroundColor: Colors.blue.shade900, child: Text('AH')),
      label: Text('Hamilton'),
    ),
    Chip(
      avatar: CircleAvatar(
          backgroundColor: Colors.blue.shade900, child: Text('ML')),
      label: Text('Lafayette'),
    ),
    Chip(
      avatar: CircleAvatar(
          backgroundColor: Colors.blue.shade900, child: Text('HM')),
      label: Text('Mulligan'),
    ),
    Chip(
      avatar: CircleAvatar(
          backgroundColor: Colors.blue.shade900, child: Text('JL')),
      label: Text('Laurens'),
    ),
  ],
)
```

## PageView Android 下的 ViewPager

估计是开发中最常用到的控件了，具体效果可以看 Demo

在 Flutter 中，通过 PageView+BottomNavigationBar 实现

```dart
body: PageView( // 类似ViewPager
  physics: NeverScrollableScrollPhysics(), // 禁止左右滑动
  controller: _controller,
  children: <Widget>[ // 类似Fragment
    _DemoHomePageView(),
    _DemoHomeMyPageView(),
  ],
  onPageChanged: (index) {
    setState(() {
      _selectedIndex = index; // 改变_selectedIndex是为了更新BottomNavigationBar的选中状态
    });
  },
)

bottomNavigationBar: BottomNavigationBar(
  items: const <BottomNavigationBarItem>[
    BottomNavigationBarItem(
      icon: Icon(Icons.home),
      title: Text('Home'),
    ),
    BottomNavigationBarItem(
      icon: Icon(Icons.person),
      title: Text('My'),
    ),
  ],
  currentIndex: _selectedIndex,
  selectedItemColor: Colors.amber[800],
  onTap: (index) {
    setState(() {
      _selectedIndex = index;
    });
    _controller.jumpToPage(index); // 类似ViewPager.setCurrentPosition
  },
)
```

> PageView 是默认不保活的，可以将\_DemoHomePageView 混入 AutomaticKeepAliveClientMixin 并保证 build 方法调用 super

> 混入 mixin 是什么，可以看 dart 语法，不同于 java 单继承，dart 的 mixin 可以扩展属性和方法

## Texture 相机

像相机这种需要实时传输大量图像数据的，Flutter 提供了 Texture，通过映射共享数据的工具

```dart
const Texture({
  Key key,
  @required this.textureId, // 通过该id获得共享的图像数据
  this.filterQuality = FilterQuality.low,
})
```

下面是一个相机例子

```dart
class _TextureBodyState extends State<TextureBody> {
  List<CameraDescription> cameras; // 可用的相机集合
  CameraController cameraController; // 相机控制中枢

  bool get isInit => (cameraController?.value?.isInitialized == true);

  /// 生成一个文件路径
  Future<String> get tempFilePath async {
    final cacheDir = await getTemporaryDirectory();
    final dirPath = '${cacheDir.path}/Picture';
    await Directory(dirPath).create(recursive: true);
    return '$dirPath/${DateTime.now().millisecondsSinceEpoch.toString()}.jpg';
  }

  String latestPicturePath;

  @override
  void initState() {
    super.initState();
    _fetchCameras();
  }

  _fetchCameras() async {
    cameras = await availableCameras();
    if (cameras.isNotEmpty) {
      cameraController = CameraController(cameras[0], ResolutionPreset.medium);
      await cameraController.initialize();
      if (mounted) {
        setState(() {});
      }
    }
  }

  @override
  void dispose() {
    cameraController?.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (!isInit) {
      return Container(
        child: Center(
          child: CircularProgressIndicator(),
        ),
      );
    }
    return Container(
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          AspectRatio(
            aspectRatio: cameraController.value.aspectRatio,
            child: CameraPreview(cameraController),
          ),
          Expanded(
            child: Row(
              children: [
                Expanded(
                  child: Center(
                    child: latestPicturePath != null
                        ? SizedBox(
                            width: 80,
                            height: 160,
                            child:
                                Image.file(File(latestPicturePath)).preview(),
                          )
                        : null,
                  ),
                ),
                Expanded(
                  child: Center(
                    child: IconButton(
                      icon: Icon(Icons.camera_alt),
                      onPressed: isInit
                          ? () async {
                              if (cameraController.value.isTakingPicture) {
                                return;
                              }
                              final filePath = await tempFilePath;
                              await cameraController.takePicture(filePath); // 拍照
                              if (mounted) {
                                setState(() {
                                  latestPicturePath = filePath;
                                });
                              }
                            }
                          : null,
                    ),
                  ),
                ),
                Spacer(),
              ],
            ),
          ),
        ],
      ),
    );
  }
}
```

## PlatformView WebView

像 WebView 这种极其复杂的原生控件，用 Flutter 重新写不太现实，所以 Flutter 也提供这样一种 Widget，将原生控件直接嵌入 Flutter 的 Widget 树。

官方已经提供 WebView 库帮我们代理了原生 WebView

```yaml
webview_flutter: ^0.3.22+1
```

```dart
WebView(
    initialUrl: widget.url,
    javascriptMode: JavascriptMode.unrestricted, // 支持js
    onWebViewCreated: (webViewController) {
      _controller.complete(webViewController);
    },
    javascriptChannels: <JavascriptChannel>[
      _toasterJavascriptChannel(context), // js通信
    ].toSet(),
    onPageStarted: (String url) {
      print('Page started loading: $url');
    },
    onPageFinished: (String url) {
      print('Page finished loading: $url');
    },
  ),
)
```
