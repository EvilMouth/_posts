---
layout: post
title: Flutter InheritedWidget
date: 2020-09-04 17:03:01
updated: 2020-09-04 17:03:01
tags:
  - flutter
  - widget
  - source
categories: Flutter
---

分析 Flutter 非常特殊的 InheritedWidget，仅次于 StatelessWidget 和 StatefulWidget

## InheritedWidget 介绍

> Base class for widgets that efficiently propagate information down the tree.

这是官方对 InheritedWidget 的介绍，大致意思就是使用 InheritedWidget 可以有效的将数据向下传播，也就是该 Widget 的子树都可以拿到这些数据。

[放个官方介绍视频](https://www.youtube.com/watch?v=Zbm3hjPjQMk)

## InheritedWidget 如何向下传播数据（共享数据）

要了解这个过程，先反向分析。

根据上面视频了解到 Theme.of(context) 获取主题就是利用 InheritedWidget 的特性，那就从 of 开始分析

```dart
static ThemeData of(BuildContext context, { bool shadowThemeOnly = false }) {
  final _InheritedTheme inheritedTheme = context.dependOnInheritedWidgetOfExactType<_InheritedTheme>();
  /// 省略省略省略
  return inheritedTheme.theme.data;
}
```

> 给需要共享数据的 Widget 提供一个 of 方法是一种常用做法，在代码阅读上非常友好

Theme Widget 通过 InheritedTheme（继承于 InheritedWidget）去共享 ThemeData，具体则是利用 BuildContext.dependOnInheritedWidgetOfExactType 实现。

> dependOnInheritedWidgetOfExactType 会使 子Widget 在 ThemeData 数据变动时**重新构建**

再继续往下看 dependOnInheritedWidgetOfExactType 做了什么，BuildContext.dependOnInheritedWidgetOfExactType 是一个空方法，具体在 Element 实现功能。

```dart
/// 存储共享元素
Map<Type, InheritedElement> _inheritedWidgets;
/// 存储依赖项
Set<InheritedElement> _dependencies;

@override
T dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({Object aspect}) {
  assert(_debugCheckStateIsActiveForAncestorLookup());

  /// 通过实现存储在 Map 的 _inheritedWidgets 上取得共享元素 InheritedElement
  final InheritedElement ancestor = _inheritedWidgets == null ? null : _inheritedWidgets[T];
  if (ancestor != null) {
    assert(ancestor is InheritedElement);
    return dependOnInheritedElement(ancestor, aspect: aspect) as T;
  }
  _hadUnsatisfiedDependencies = true;
  return null;
}

@override
InheritedWidget dependOnInheritedElement(InheritedElement ancestor, { Object aspect }) {
  assert(ancestor != null);

  /// 将共享数据元素添加为依赖项，所以当依赖变动时，当前元素会更新，从而更新状态
  _dependencies ??= HashSet<InheritedElement>();
  _dependencies.add(ancestor);
  ancestor.updateDependencies(this, aspect);

  /// 将该共享元素 InheritedElement 对应的 InheritedWidget 返回
  /// 也就是在上面 Theme.of(context) 中拿到的 InheritedTheme
  return ancestor.widget;
}
```

通过上面的源码可以看到有两个重要的成员 _inheritedWidgets 以及 _dependencies。_dependencies 涉及的是依赖更新，不是当前篇章的内容，_inheritedWidgets 才是能够**向下传递数据**的关键。

先回想以下 Theme.of(context) 中的 context 是什么

在 子Widget 中调用 Theme.of(context) 中，context 指的就是 当前子Widget对应的Element，也就是说 context.dependOnInheritedWidgetOfExactType 就是对 当前子Element 的调用，那自然 _inheritedWidgets 就是 当前子Element 的成员。

> 向下传递就是让所以 子Widget 都持有 祖先共享数据 的一份拷贝指向，使得 子Widget 在获取 祖先共享数据 时十分方便快捷

那为何 当前子Element 的 _inheritedWidgets 会有 **祖先** 的数据呢，答案就在 Element 本身一个方法上

```dart
/// Element 的实现
void _updateInheritance() {
  assert(_active);
  _inheritedWidgets = _parent?._inheritedWidgets;
}

/// InheritedElement 的实现
@override
void _updateInheritance() {
  assert(_active);
  final Map<Type, InheritedElement> incomingWidgets = _parent?._inheritedWidgets;
  if (incomingWidgets != null)
    _inheritedWidgets = HashMap<Type, InheritedElement>.from(incomingWidgets);
  else
    _inheritedWidgets = HashMap<Type, InheritedElement>();
  _inheritedWidgets[widget.runtimeType] = this;
}
```

- _updateInheritance 会在 Element 处于激活状态下被调用
- _updateInheritance 会将 父Element 的 _inheritedWidgets 赋值给 当前Element 的 _inheritedWidgets
- InheritedElement 重写 _updateInheritance，并将自己写进 _inheritedWidgets
- 以此完成 向下传递、数据共享

## _dependencies 的作用

回到刚才两个重要成员中的另一个成员 _dependencies，_dependencies 起到的作用是当 祖先共享数据 发现变化时，子Widget 就会更新的作用

```dart
_dependencies ??= HashSet<InheritedElement>();
_dependencies.add(ancestor);
ancestor.updateDependencies(this, aspect);
```

- _dependencies.add(ancestor); 也就是将 祖先元素 作为依赖项存储
- ancestor.updateDependencies(this, aspect); 则是 祖先元素 将 子元素 作为依赖者存储
- 两者互相持有引用

当 祖先元素共享数据 变化时，祖先元素会 rebuild，再而通过 Flutter Framework 调度走到 update() 方法

```dart
/// InheritedElement 的成员
final Map<Element, Object> _dependents = HashMap<Element, Object>(); // 存储依赖者

@override
void updated(InheritedWidget oldWidget) {
  if (widget.updateShouldNotify(oldWidget)) // 判断是否需要更新
    super.updated(oldWidget);
}

@protected
void updated(covariant ProxyWidget oldWidget) {
  notifyClients(oldWidget); // 通知依赖者
}

@override
void notifyClients(InheritedWidget oldWidget) {
  assert(_debugCheckOwnerBuildTargetExists('notifyClients'));
  for (final Element dependent in _dependents.keys) { // 循环依赖者
    assert(() {
      // check that it really is our descendant
      Element ancestor = dependent._parent;
      while (ancestor != this && ancestor != null)
        ancestor = ancestor._parent;
      return ancestor == this;
    }());
    // check that it really depends on us
    assert(dependent._dependencies.contains(this));
    notifyDependent(oldWidget, dependent); // 通知依赖者
  }
}

@protected
void notifyDependent(covariant InheritedWidget oldWidget, Element dependent) {
  dependent.didChangeDependencies(); // 调用依赖者的 didChangeDependencies
}

@mustCallSuper
void didChangeDependencies() {
  assert(_active); // otherwise markNeedsBuild is a no-op
  assert(_debugCheckOwnerBuildTargetExists('didChangeDependencies'));
  markNeedsBuild(); // 重新构建
}
```

> 一开始 InheritedWidget 没有任何 依赖者，当有 子Widget 请求了数据（Theme.of）之后，便使得 祖先元素 和 子元素 互相绑定

> 其实还有一种获取 祖先共享数据 的方法，并且不产生依赖关系 - BuildContext.getElementForInheritedWidgetOfExactType

## aspect 的作用

```dart
T dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({ Object aspect });
InheritedWidget dependOnInheritedElement(InheritedElement ancestor, { Object aspect });
```

再回顾 dependOnInheritedWidgetOfExactType、dependOnInheritedElement 有个 aspect 可选参数，具体是什么作用

> aspect 提供颗粒化共享数据

假如一个祖先提供A、B两个数据，他有两个孩子，孩子C引用数据A，孩子D引用数据B。当只有数据A发生变化时，其实只需要重新构建孩子C，aspect就是提供这么一个过滤功能

aspect 在 InheritedWidget 并没有使用，而是在 InheritedModel 作用体现，InheritedModel 可以通过 [官方视频](https://www.youtube.com/watch?v=ml5uefGgkaA) 了解

在上面分析可得知 建立依赖关系 是双向的，祖先元素的 updateDependencies(Element dependent, Object aspect) 会被调用，在 InheritedModel 中重写了该方法

```dart
@override
void updateDependencies(Element dependent, Object aspect) {
  final Set<T> dependencies = getDependencies(dependent) as Set<T>;
  if (dependencies != null && dependencies.isEmpty)
    return;

  if (aspect == null) {
    setDependencies(dependent, HashSet<T>());
  } else {
    assert(aspect is T);
    setDependencies(dependent, (dependencies ?? HashSet<T>())..add(aspect as T));
  }
}
```

如果传了 aspect，则将其放入 HashSet 中放入 dependencies，并且已知 祖先更新会通知子元素（调用 notifyDependent），InheritedModel 也重写了该方法

```dart
@override
void notifyDependent(InheritedModel<T> oldWidget, Element dependent) {
  final Set<T> dependencies = getDependencies(dependent) as Set<T>;
  if (dependencies == null)
    return;
  if (dependencies.isEmpty || widget.updateShouldNotifyDependent(oldWidget, dependencies))
    dependent.didChangeDependencies();
}

/// 具体实现，通过 aspects.contains('a')) 是否包含 a或b 去过滤
@override
bool updateShouldNotifyDependent(ABModel old, Set<String> aspects) {
  return (a != old.a && aspects.contains('a'))
      || (b != old.b && aspects.contains('b'))
}
```

最后在 子Widget 获取 祖先共享数据 时需要通过 InheritedModel.inheritFrom 去获取

```dart
class ABModel extends InheritedModel<String> {
  static of(BuildContext context, String aspect) {
    return InheritedModel.inheritFrom<MyModel>(context, aspect: aspect);
  }
}

ABModel.of(context, 'a');
ABModel.of(context, 'b');
```

> 官方状态管理库 Provider 就是利用 InheritedWidget 的特性实现
