---
layout: post
title: Fun Flutter Widget
date: 2020-08-14 11:18:09
updated: 2020-08-14 11:18:09
tags: widget
categories: Flutter
---

本文介绍日常开发中比较实用的Flutter Widget

## Placeholder 占位图

常用在异步加载请求时先使用Placeholder代替显示，请求结束后替换

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

例如登陆页面，利用Form Widget将用户名和密码两个输入框包起来，通过Key进行统一校验，还自带返回按键监听

```dart
const Form({
  Key key,
  @required this.child,
  this.autovalidate = false, // 自动校验
  this.onWillPop, // 返回按键监听
  this.onChanged, // 表单内容变化
})
```

> 在Form里使用的输入框目前只提供TextFormField和DropdownButtonFormField，两者都继承FormField，可自行扩展

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

### FormField为何要被包在Form里面

Form是个StatefuleWidget，state是FormState，FormState提供了一些操作FormField的方法，比如validate、save。如何获取呢，原因在FormField的build()方法内向上获取到Form，并将自己注册给Form

```dart
@override
Widget build(BuildContext context) {
  Form.of(context)?._register(this); // 将自己注册给Form
  return widget.builder(this);
}
```

> Form.of是什么，这就要讲到Flutter的InheritedWidget，InheritedWidget是一个方便子Widget获取父Widget提供的属性的Widget。Theme.of、Local.of都是基于InheritedWidget

## FutureBuilder 支持异步加载的Widget

一般如何实现异步加载动态更新布局，自然是通过setState改变状态。FutureBuilder就是这么一个封装的Widget，应用在比较单一、一次性的场景

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

> builder会回调多次，原因就是snapshot是一个状态，会改变多次

## WillPopScope 监听Android返回按钮

Android才需要的返回按键监听，例如应用在双击返回键退出应用等操作

> 上面介绍的Form也有一个onWillPop属性，其实就是包了一层WillPopScope

```dart
const WillPopScope({
  Key key,
  @required this.child,
  @required this.onWillPop, // Future<bool> Function()
})
```

下面的例子通过WillPopScope包裹AlertDialog来实现一个无法通过返回按键取消的弹窗，只需要onWillPop返回flase即表示屏蔽返回

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

> Flutter的路由实际就是用Overlay实现
> 
> 可以理解为Flutter的世界只有一个Activity，不同的Page其实就是Fragment堆叠
> 
> 所以在Flutter实现全局悬浮按钮简直不要太简单

> Overlay跟着Navigator一起被创建，所以通过直接Overlay.of取得OverlayState

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
> 利用insert和remove即可实现Flutter层面的Toast

## AbsorbPointer 快速禁止屏幕点击

通过AbsorbPointer包裹住RaisedButton，即使RaisedButton声明了点击事件，但是AbsorbPointer也声明了absorbing: true，所有RaisedButton是点不了的

```dart
AbsorbPointer(
  absorbing: true, // 是否吸收触摸事件
  child: RaisedButton(onPressed: () {}),
)
```

> Navigator其实也包裹了一层AbsorbPointer，在路由跳转过程，会吸收所有屏幕触摸事件，让跳转完成

## Dismissible 列表Item滑动移除

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

在Android 21以上，可以通过给View设置一个flag来让页面跳转时两个View直接有个过渡效果，通常用于浏览大图。在Flutter也是类似

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

> 只需要将Image包裹在Hero中并提供一个tag，在第二个页面同样标记，剩下的就交给Flutter帮我们实现过渡

## FractionallySizedBox 屏幕百分比

通常用在某些Widget需要撑满屏幕

```dart
const FractionallySizedBox({
  Key key,
  this.alignment = Alignment.center,
  this.widthFactor, // 0-1 1表示撑满
  this.heightFactor,
  Widget child,
})
```

## SizeBox 控制Widget大小

字面意思，向父Widget申请一定大小的空间

```dart
SizeBox(width: 10, height: 10, child: null)
```

- 充当两个Widget直接的Margin
- 控制Widget大小

## Transform 改变Widget形态

与Android的作用一致，可以对Widget进行形态上的改变，比如伪3D旋转

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

## IndexStack 快速切换Widget

例如某个Widget拥有很多种状态，每个状态不同的Icon，通过IndexStack先将所有Icon提供，通过改变index快速切换Icon

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

> IndexedStack继承Stack，并重写了其渲染部分，只渲染index命中的Icon

## Wrap 流式布局

> Wrap基本与Row/Column一致，不同的一点是当children超出范围的话，Wrap会自动将超出的chilren换行

下面的例子中第三第四个Chip后被layout在第二行

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

## PageView Android下的ViewPager

估计是开发中最常用到的控件了，具体效果可以看Demo

在Flutter中，通过PageView+BottomNavigationBar实现

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

> PageView是默认不保活的，可以将_DemoHomePageView混入AutomaticKeepAliveClientMixin并保证build方法调用super

> 混入mixin是什么，可以看dart语法，不同于java单继承，dart的mixin可以扩展属性和方法

## Texture 相机

像相机这种需要实时传输大量图像数据的，Flutter提供了Texture，通过映射共享数据的工具

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

像WebView这种极其复杂的原生控件，用Flutter重新写不太现实，所以Flutter也提供这样一种Widget，将原生控件直接嵌入Flutter的Widget树。

官方已经提供WebView库帮我们代理了原生WebView

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
