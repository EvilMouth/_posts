---
layout: post
title: Flutter State
date: 2020-08-14 11:20:45
updated: 2020-08-14 11:20:45
tags:
  - flutter
  - state
  - source
categories: Flutter
---

Flutter 状态管理介绍

## State 是什么

### 与 Android 不同之处

在 Android，比如想要动态改变一个 TextView 的 text，则需要在通过 id 获取 TextView，并通过 TextView 提供的 setText(newText)方法设置新的 text，随后重新渲染

```java
TextView tv = findViewById(R.id.tv);
tv.setText("newText");
```

而在 Flutter 中，则是通过 setState()去改变

```dart
String text = 'text';

Text(text); // Text Widget引用text变量

setState(() {
  text = 'newText'; // 在setState中改变text变量
});
```

### setState 发生了什么

```dart
@protected
void setState(VoidCallback fn) {
  // 省略一堆assert
  fn();
  _element.markNeedsBuild(); // 将当前Element标记为dirty
}
```

setState 只是简单的标记一个 dirty（而不是 Android View 主动刷新），真正发起重新渲染的是 Flutter 的渲染机制，简单流程如下

- Flutter Engine 接收到 Vsync 垂直同步信号
- 扫描 Element 树有没有 dirty 标记的 Element
- rebuild 该 element

> Element 由 Widget 生成，每一个 Widget 对应一个 Element

那是否可以像下面这样写，完全可以。但这是一种规范，可以一眼看出哪些状态会被变更

```dart
text = 'newText';
setState(() {});
```

### 思考

从上面的例子可以看出 Flutter 是以状态管理的方式，通过改变状态从而改变 UI，这自然引出一个刷新 UI 效率的问题，因为不像 Android 通过精准的改变一个目标 View 去刷新

## Element 是什么

实际 Flutter 开发页面操作的各种 Widget，都会各自生成对应的 Element，Element 又会根据需要生成 RenderObject，所以 Flutter 会生成三棵树

> Widget 树 -> Element 树 -> RenderObject 树

- Widget 树就是我们代码层面生成的，Widget 只是一个配置文件
- Element 树则是根据 Widget 树一一对应生成的，主要功能就是管理复用
- RenderObject 树才是实实在在的渲染到屏幕

### Element 的复用

众所周知在 build(context)方法返回 Widget，而 build 会在状态变化时重新被调用，也就是 Widget 会被不断的重新生成，如果渲染依赖的是 Widget，Widget 又持有 RenderObject 也就是渲染数据，那将是大量的内存浪费。

所以 Widget 只是一个配置文件，存储的是一些属性，而 Element 才是真正持有 RenderObject 的对象

在触发 rebuild 时，Element 的 update()会被调用，update()会调用 canUpdate()来判断是否可以复用 Widget，如果可以复用，则只是更新配置

```dart
static bool canUpdate(Widget oldWidget, Widget newWidget) {
  return oldWidget.runtimeType == newWidget.runtimeType
      && oldWidget.key == newWidget.key;
}
```

## 如何控制刷新范围

Flutter 内部已经尽可能的优化了渲染流程，但是在实际开发中，依然要注意刷新范围，总结一个点就是

> setState 要写在哪里

### setState 刷新的是哪块区域

再翻出 setState 的代码

```dart
@protected
void setState(VoidCallback fn) {
  // 省略一堆assert
  fn();
  _element.markNeedsBuild(); // 将当前Element标记为dirty
}
```

上面已经介绍到 Flutter Engine 会 rebuild 被标记为 dirty 的 element，那在上面 setState 的代码可以看到，就是当前的 element 被标记为 dirty，也就是说

> 在哪个 Element 调用 setState，那这个 Element 包括就会 rebuild

### 刷新影响

简单看官方 Counter 例子，该例子发生变化的其实只是某个 Text，但是在下面两处注释可以看到实际上整个页面都会 rebuild。

> 当页面比较复杂或者像 FutureBuilder 这种类型 Widget 时，就要尽可能控制 setState 位置

> 不过通过该例子并不是说这样写是错误的，还记得 Element 的复用逻辑吗，虽然该例子整个 Scaffold 以及子 Widget 都会重新创建，但是实际变化的 Element 只是其中引用了\_counter 的 Text 而已

```dart
class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);

  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;

  void _incrementCounter() {
    setState(() { // 调用setState的Element是MyHomePage的Element
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) { // 所以该build方法在每次increment都会调用一次
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              'You have pushed the button this many times:',
            ),
            Text(
              '$_counter',
              style: Theme.of(context).textTheme.headline4,
            ),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _incrementCounter,
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
    );
  }
}
```

### 如何控制

#### 方式一

首先 Flutter 是组合 Widget 的编写方式，StatelessWidget 嵌套 StatefulWidget 再嵌套 StatelessWidget，需要状态改变的地方可以重新封装成一个 StatefulWidget 从而缩小 setState 影响范围

> 下面例子通过将 Text 和 Button 封装到 IncrementWidget 中，此时调用 setState 的 Element 只是 IncrementWidget 的 Element，所以控制了刷新范围

```dart
class IncrementWidget extends StatefulWidget {
  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _IncrementState extends State<IncrementWidget> {
  int _counter = 0;

  void _incrementCounter() {
    setState(() {
      _counter++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      mainAxisAlignment: MainAxisAlignment.center,
      children: <Widget>[
        Text(
          'You have pushed the button this many times:',
        ),
        Text(
          '$_counter',
          style: Theme.of(context).textTheme.headline4,
        ),
        RaisedButton(
          onPressed: _incrementCounter,
          child: Icon(Icons.add),
        ),
      ],
    );
  }
}
```

#### 方式二

方式一的例子只是简单的维护一个 int 类型的状态，当遇到状态更为复杂的情况，比如该状态不仅仅一个地方引用到，在别的地方也需要控制，那自然无法只是封装一个 Widget 就能搞定的。再拿官方 Counter 作为例子

> 下面的例子的做法并不推荐，中心思想依然是想表达如何缩小 setState 范围

```dart
class MyHomePage extends StatelessWidget {
  MyHomePage({Key key, this.title}) : super(key: key);

  final String title;
  int _counter = 0;
  StateSetter _setState; // 获取Text的setState

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Text(
              'You have pushed the button this many times:',
            ),
            StatefulBuilder(builder: (context, setState) {
              _setState = setState;
              return Text(
                '$_counter',
                style: Theme.of(context).textTheme.headline4,
              );
            }),
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          _setState(() { // 使用的是Text的setState
            _counter++;
          });
        },
        tooltip: 'Increment',
        child: Icon(Icons.add),
      ),
    );
  }
}
```

## Provider 框架

当业务逻辑越来越复杂，状态管理自然成为至关重要的一个门槛，社区也开源了许多状态管理的框架，例如 Bloc、Redux

这里主要介绍官方 Provider 使用以及注意事项

### Provider 使用

首先引入 Provider 库

```yaml
provider: ^4.3.1
```

官方的 Counter 例子可以这样写，几个重要点

- ChangeNotifier#notifyListeners()
- ChangeNotifierProvider#create()
- context.watch<Counter>()
- context.read<Counter>()

```dart
class Counter with ChangeNotifier {
  int _count = 0;
  int get count => _count;

  void increment() {
    _count++;
    notifyListeners(); // 通知监听者
  }
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      home: ChangeNotifierProvider( // 通过Provider包裹child
        create: (context) => Counter(), // 创建Counter，可以理解为ViewModel
        child: MyHomePage(),
      ),
    );
  }
}

class MyHomePage extends StatelessWidget {
  const MyHomePage({Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Example'),
      ),
      body: Center(
        child: Column(
          mainAxisSize: MainAxisSize.min,
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            const Text('You have pushed the button this many times:'),
            Text('${context.watch<Counter>().count}'), // 通过watch监听count变化
          ],
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => context.read<Counter>().increment(), // 通过read获取Counter对象调用increment方法
        tooltip: 'Increment',
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

> context.watch 和 context.read 是通过 dart 的 Extension 语法扩展
>
> watch 会开启监听，当发生变化时，context（也就是 Element）就会 rebuild
>
> 还有个 context.select<T, R>()允许监听部分数据

### Consumer

上面的例子同样会看到一个问题，就是 watch 调用的 context 是整个 MyHomePage，所以当 count 变化时整个 MyHomePage 会 rebuild，依然浪费了不必要的 Widget 创建，所以 Provider 提供了 Consumer

下面例子通过 Consumer 包裹 Text，其实也就是封装了一个通用的 Widget，为的就是缩小刷新范围

```dart
children: <Widget>[
  const Text('You have pushed the button this many times:'),
  Consumer<Counter>(
    builder: (context, counter, child) => Text('${counter.count}')
  ),
],
```

### Selector

页面再复杂一点，看这么一个例子。

一个评论列表，每个评论都有个点赞按钮，这里就有个注意点 - _局部刷新_。

```dart
class Comment {
  String content;
  bool like; // 是否点赞

  Comment(this.content, this.like);
}

class CommentListProvider with ChangeNotifier {
  List<Comment> _commentList =
      List.generate(5, (index) => Comment('Comment $index', false));

  get commentList => _commentList;

  get total => _commentList.length;

  like(int index) {
    var comment = _commentList[index];
    _commentList[index] = Comment(comment.content, !comment.like);
    notifyListeners();
  }
}

ChangeNotifierProvider(
  create: (_) => CommentListProvider(),
  child: Selector<CommentListProvidCommentListProvider>( // 1.Selector包住ListView而不是用Consumer
    selector: (context, provider) => provider,
    builder: (context, provider, child) {
      return ListView.builder(
        physics: NeverScrollableScrollPhysics(),
        itemCount: provider.total,
        itemBuilder: (context, index) {
          return Selector<CommentListProvidComment>( // Selector包住每个Item
            selector: (context, provider) =>
                provider.commentList[index], // 2.selector返回单个Comment
            builder: (context, comment, child) {
              return ListTile(
                title: Text(comment.content),
                trailing: IconButton(
                  color: Colors.accents[
                      Random().nextInt(Colaccents.length)],
                  icon: Icon(comment.like
                      ? Icons.star
                      : Icons.star_border),
                  onPressed: () {
                    provider.like(index); // 点赞
                  },
                ),
              );
            },
          );
        },
      );
    },
  ),
)
```

重点讲解

1. Selector 与 Consumer 唯一区别是 Selector Widget 会缓存当前 Widget，并在 rebuild 的时候判断差异才去更新
2. 第二个 Selector 包住每个 Item，当状态变化时，Selector 只会识别到当前的评论数据发生了变化，所以就只会刷新该 Item

看下 Selector 的 build 方法

```dart
T value; // 缓存数据，该例子指的是Comment
Widget cache; // 缓存Widget
Widget oldWidget; // 缓存Selector Widget

@override
Widget buildWithChild(BuildContext context, Widget child) {
  final selected = widget.selector(context);

  var shouldInvalidateCache = oldWidget != widget ||
      (widget._shouldRebuild != null &&
          widget._shouldRebuild.call(value, selected)) ||
      (widget._shouldRebuild == null &&
          !const DeepCollectionEquality().equals(value, selected); // DeepCollectionEquality判断新旧数据的不同，也可定义shouldRebuild覆盖该默认判断逻辑
  if (shouldInvalidateCache) {
    value = selected;
    oldWidget = widget;
    cache = widget.builder( // 这里就是构建了ListTile
      context,
      selected,
      child,
    );
  }
  return cache;
}
```

### Consumer 与 Selector 区别

> Consumer - 字面意思消费者，故只要状态变更，就会 rebuild
> 而 Selector 内部存在缓存，视具体情况 rebuild

### 更多 Provider

| name                                                                                                                          | description                                                                                                                                                            |
| ----------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [Provider](https://pub.dartlang.org/documentation/provider/latest/provider/Provider-class.html)                               | The most basic form of provider. It takes a value and exposes it, whatever the value is.                                                                               |
| [ListenableProvider](https://pub.dartlang.org/documentation/provider/latest/provider/ListenableProvider-class.html)           | A specific provider for Listenable object. ListenableProvider will listen to the object and ask widgets which depend on it to rebuild whenever the listener is called. |
| [ChangeNotifierProvider](https://pub.dartlang.org/documentation/provider/latest/provider/ChangeNotifierProvider-class.html)   | A specification of ListenableProvider for ChangeNotifier. It will automatically call `ChangeNotifier.dispose` when needed.                                             |
| [ValueListenableProvider](https://pub.dartlang.org/documentation/provider/latest/provider/ValueListenableProvider-class.html) | Listen to a ValueListenable and only expose `ValueListenable.value`.                                                                                                   |
| [StreamProvider](https://pub.dartlang.org/documentation/provider/latest/provider/StreamProvider-class.html)                   | Listen to a Stream and expose the latest value emitted.                                                                                                                |
| [FutureProvider](https://pub.dartlang.org/documentation/provider/latest/provider/FutureProvider-class.html)                   | Takes a `Future` and updates dependents when the future completes.                                                                                                     |
