---
layout: post
title: Flutter runApp
date: 2020-09-03 16:58:07
updated: 2020-09-03 16:58:07
tags: flutter
categories: Flutter
---

runApp过程介绍，需要先了解mixin概念

## runApp

runApp作为Flutter启动App入口，具体做了哪些操作，通过源码一步步分析

```dart
void runApp(Widget app) {
  WidgetsFlutterBinding.ensureInitialized()
    ..scheduleAttachRootWidget(app)
    ..scheduleWarmUpFrame();
}
```

- ensureInitialized 负责创建一个 WidgetsBinding 的全局单例
- scheduleAttachRootWidget 负责将 app（也就是我们创建的首个 Widget）附加到 RenderView（Flutter 渲染树的根） 上
- scheduleWarmUpFrame 则是立即请求一次绘制，将渲染树渲染出来

### ensureInitialized - WidgetsFlutterBinding 起到什么作用

先看下 WidgetsFlutterBinding 类的定义

```dart
/// A concrete binding for applications based on the Widgets framework.
///
/// This is the glue that binds the framework to the Flutter engine.
class WidgetsFlutterBinding extends BindingBase with GestureBinding, SchedulerBinding, ServicesBinding, PaintingBinding, SemanticsBinding, RendererBinding, WidgetsBinding {

  /// Returns an instance of the [WidgetsBinding], creating and
  /// initializing it if necessary. If one is created, it will be a
  /// [WidgetsFlutterBinding]. If one was previously initialized, then
  /// it will at least implement [WidgetsBinding].
  static WidgetsBinding ensureInitialized() {
    if (WidgetsBinding.instance == null)
      WidgetsFlutterBinding();
    return WidgetsBinding.instance;
  }
}
```

> WidgetsFlutterBinding 实际上只是一个粘合剂，通过继承 BindingBase 以及使用 *混入* 将一系列 Binding 结合起来。

所以具体作用还得往下看。直接看 BindingBase 的构造函数

```dart
/// Default abstract constructor for bindings.
///
/// First calls [initInstances] to have bindings initialize their
/// instance pointers and other state, then calls
/// [initServiceExtensions] to have bindings initialize their
/// observatory service extensions, if any.
BindingBase() {
  initInstances();
  initServiceExtensions();
}
```

- initInstances 是一个空实现，具体作用是让 *混入* 的一系列 Binding 进行自己的初始化，后续会介绍各自的作用
- initServiceExtensions 允许子类注册一些服务扩展监听

接下来就一个个 Binding 看下都是干什么的

#### GestureBinding 手势相关

```dart
mixin GestureBinding on BindingBase implements HitTestable, HitTestDispatcher, HitTestTarget {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    window.onPointerDataPacket = _handlePointerDataPacket;
  }
}
```

> mixin 将 GestureBinding 声明为混入类，允许其他类通过 with 将其混入并获得其能力。on 类似继承，并表明如果有类想混入 GestureBinding，该类必须先继承 BindingBase。后续的 Binding 类都是如此实现

> window 在 Flutter 是一个全局单例，是整个框架的根基，所有跟屏幕相关的事件（绘制、分发）都由其控制

GestureBinding 的初始化实现了 onPointerDataPacket 回调，负责处理窗口手势事件。并实现了 HitTest 相关的接口，目的就是为了进行命中测试查找到点击屏幕的位置对应的 Widget 然后分发事件

#### SchedulerBinding Frame调度相关

```dart
@protected
void ensureFrameCallbacksRegistered() {
  window.onBeginFrame ??= _handleBeginFrame;
  window.onDrawFrame ??= _handleDrawFrame;
}
```
SchedulerBinding 主要负责每一帧的调度，Ticker 和 Animation 就是由他所控制。

SchedulerBinding 实现了 onBeginFrame 和 onDrawFrame 回调，并处理屏幕刷新事件。handleBeginFrame 和 handleDrawFrame 就是负责执行每一帧需要处理的事务，这些事务可以通过 SchedulerBinding.instance 进行注册，其中共有三种类型的事务
- transientCallbacks 用于处理一些临时的绘制，例如动画
- persistentCallbacks 负责布局绘制工作
- postFrameCallbacks 处理一次性绘制

再回顾 runApp 的代码，其中最后一步 scheduleWarmUpFrame() 正是 SchedulerBinding 的方法实现，具体在下面篇章介绍

#### ServicesBinding 通信相关

```dart
mixin ServicesBinding on BindingBase, SchedulerBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    _defaultBinaryMessenger = createBinaryMessenger();
    window.onPlatformMessage = defaultBinaryMessenger.handlePlatformMessage;
    initLicenses();
    SystemChannels.system.setMessageHandler(handleSystemMessage);
    SystemChannels.lifecycle.setMessageHandler(_handleLifecycleMessage);
    readInitialLifecycleStateFromNativeWindow();
  }
}
```

ServicesBinding 负责与 Native 进行通信，在初始化时分别做一下操作
- createBinaryMessenger() 创建通信信使
- 实现 onPlatformMessage 回调，用以处理平台消息
- initLicenses() 添加一些许可，会自动整合然后通过 LicensePage 展示，一般不会用到
- SystemChannels.system 监听平台内存紧张回调，用于清理图片缓存、Widget一些自定义处理
- SystemChannels.lifecycle 接收平台页面生命周期事件，只有处于前台才会调度绘制
- readInitialLifecycleStateFromNativeWindow() 读取平台当前生命状态进行绘制的调度

#### PaintingBinding 字体变动、图片缓存相关

```dart
mixin PaintingBinding on BindingBase, ServicesBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    _imageCache = createImageCache();
    if (shaderWarmUp != null) {
      shaderWarmUp.execute();
    }
  }
}
```

PaintingBinding 的初始化中通过 createImageCache() 创建了一个全局图片缓存单例，Image Widget 便是将图片缓存在这里。

> shaderWarmUp 为 Skia着色器 热身，热身就是为了防止可能出现的卡顿

PaintingBinding 还实现了 ServicesBinding 的 handleSystemMessage(Object systemMessage) 来监听平台的 fontsChange 事件，可以通过 PaintingBinding.instance.systemFonts.addListener 来进行监听

#### SemanticsBinding 辅助功能相关

```dart
mixin SemanticsBinding on BindingBase {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    _accessibilityFeatures = window.accessibilityFeatures;
  }
}
```

SemanticsBinding 主要作用与一些无障碍服务上，比如图片色盲、文字朗读，在国内不受重视

#### RendererBinding 渲染相关

```dart
/// The glue between the render tree and the Flutter engine.
mixin RendererBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, SemanticsBinding, HitTestable {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;
    _pipelineOwner = PipelineOwner(
      onNeedVisualUpdate: ensureVisualUpdate,
      onSemanticsOwnerCreated: _handleSemanticsOwnerCreated,
      onSemanticsOwnerDisposed: _handleSemanticsOwnerDisposed,
    );
    window
      ..onMetricsChanged = handleMetricsChanged
      ..onTextScaleFactorChanged = handleTextScaleFactorChanged
      ..onPlatformBrightnessChanged = handlePlatformBrightnessChanged
      ..onSemanticsEnabledChanged = _handleSemanticsEnabledChanged
      ..onSemanticsAction = _handleSemanticsAction;
    initRenderView();
    _handleSemanticsEnabledChanged();
    assert(renderView != null);
    addPersistentFrameCallback(_handlePersistentFrameCallback);
    initMouseTracker();
  }
}
```

> RendererBinding 官方定义为渲染树与 Flutter engine 的粘合剂，负责渲染树的一切行为

通过 RendererBinding 的初始化可以看到做了这些操作
- PipelineOwner 的创建，PipelineOwner 负责渲染一系列流程的工作，setState 后的 dirty 状态就是由他维护
- 实现了 onMetricsChanged 回调，处理缩放比例改变并通过 SchedulerBinding 调度渲染
- 实现了 onTextScaleFactorChanged 回调，处理文字缩放改变，通过 WidgetsBinding.instance.addObserver 添加监听
- 实现了 onPlatformBrightnessChanged 回调，处理亮度改变，一样是通过 WidgetsBinding 去添加监听
- 实现了 semantic 相关回调，处理辅助服务相关内容
- initRenderView() 创建渲染树的根元素
- addPersistentFrameCallback(_handlePersistentFrameCallback) 向 SchedulerBinding 的 _persistentCallbacks 注册一个回调，该回调负责的就是渲染流程的动画、计算、布局、绘制的工作
- initMouseTracker() 创建焦点追踪器，在例如 Tooltip Widget、高亮中就使用到

#### WidgetsBinding Widget相关

```dart
/// The glue between the widgets layer and the Flutter engine.
mixin WidgetsBinding on BindingBase, ServicesBinding, SchedulerBinding, GestureBinding, RendererBinding, SemanticsBinding {
  @override
  void initInstances() {
    super.initInstances();
    _instance = this;

    assert(() {
      _debugAddStackFilters();
      return true;
    }());

    // Initialization of [_buildOwner] has to be done after
    // [super.initInstances] is called, as it requires [ServicesBinding] to
    // properly setup the [defaultBinaryMessenger] instance.
    _buildOwner = BuildOwner();
    buildOwner.onBuildScheduled = _handleBuildScheduled;
    window.onLocaleChanged = handleLocaleChanged;
    window.onAccessibilityFeaturesChanged = handleAccessibilityFeaturesChanged;
    SystemChannels.navigation.setMethodCallHandler(_handleNavigationInvocation);
    FlutterErrorDetails.propertiesTransformers.add(transformDebugCreator);
  }
}
```

> WidgetsBinding 是所有 Binding 中最重要的一环，**Flutter 万物皆 Widget**，WidgetsBinding 负责

WidgetsBinding 初始化主要做了以下操作
- 创建 BuildOwner，负责管控那些需要**rebuild**的 widget（也就是标记为 dirty 的 element）
- handleBuildScheduled 会在 widget rebuild 时调用，并发起绘制调度
- 实现 onLocaleChanged 回调，处理语言变化
- handleNavigationInvocation 处理路由的入栈和出栈

### scheduleAttachRootWidget - RenderView 是什么

大致了解各个 Binding 的主要作用后，会了解到其中最重要的一环 WidgetsBinding 以及渲染树的根 RenderView，下面就从 runApp 的第二步调用介绍起

```dart
/// Schedules a [Timer] for attaching the root widget.
///
/// This is called by [runApp] to configure the widget tree. Consider using
/// [attachRootWidget] if you want to build the widget tree synchronously.
@protected
void scheduleAttachRootWidget(Widget rootWidget) {
  Timer.run(() {
    attachRootWidget(rootWidget);
  });
}

/// Takes a widget and attaches it to the [renderViewElement], creating it if
/// necessary.
///
/// This is called by [runApp] to configure the widget tree.
///
/// See also:
///
///  * [RenderObjectToWidgetAdapter.attachToRenderTree], which inflates a
///    widget and attaches it to the render tree.
void attachRootWidget(Widget rootWidget) {
  _readyToProduceFrames = true;
  _renderViewElement = RenderObjectToWidgetAdapter<RenderBox>(
    container: renderView,
    debugShortDescription: '[root]',
    child: rootWidget,
  ).attachToRenderTree(buildOwner, renderViewElement as RenderObjectToWidgetElement<RenderBox>);
}
```

scheduleAttachRootWidget 是 WidgetsBinding 的一个成员方法，负责将我们的 app widget 附加到 RenderView 上。通过 attachRootWidget 可以看到创建了 _renderViewElement，可以理解为 RenderView 对应的 Element。最后根据 app widget 所形成的 widget tree 创建出对应的 element tree。

### scheduleWarmUpFrame - 做什么的

runApp 的最后一步调用，当前两步完成了各种绑定以及渲染树的生成后，SchedulerBinding#scheduleWarmUpFrame 被调用。

```dart
/// Schedule a frame to run as soon as possible, rather than waiting for
/// the engine to request a frame in response to a system "Vsync" signal.
///
/// This is used during application startup so that the first frame (which is
/// likely to be quite expensive) gets a few extra milliseconds to run.
///
/// Locks events dispatching until the scheduled frame has completed.
///
/// If a frame has already been scheduled with [scheduleFrame] or
/// [scheduleForcedFrame], this call may delay that frame.
///
/// If any scheduled frame has already begun or if another
/// [scheduleWarmUpFrame] was already called, this call will be ignored.
///
/// Prefer [scheduleFrame] to update the display in normal operation.
void scheduleWarmUpFrame() {
  if (_warmUpFrame || schedulerPhase != SchedulerPhase.idle)
    return;

  _warmUpFrame = true;
  Timeline.startSync('Warm-up frame');
  final bool hadScheduledFrame = _hasScheduledFrame;
  // We use timers here to ensure that microtasks flush in between.
  Timer.run(() {
    assert(_warmUpFrame);
    handleBeginFrame(null);
  });
  Timer.run(() {
    assert(_warmUpFrame);
    handleDrawFrame();
    // We call resetEpoch after this frame so that, in the hot reload case,
    // the very next frame pretends to have occurred immediately after this
    // warm-up frame. The warm-up frame's timestamp will typically be far in
    // the past (the time of the last real frame), so if we didn't reset the
    // epoch we would see a sudden jump from the old time in the warm-up frame
    // to the new time in the "real" frame. The biggest problem with this is
    // that implicit animations end up being triggered at the old time and
    // then skipping every frame and finishing in the new time.
    resetEpoch();
    _warmUpFrame = false;
    if (hadScheduledFrame)
      scheduleFrame();
  });

  // Lock events so touch events etc don't insert themselves until the
  // scheduled frame has finished.
  lockEvents(() async {
    await endOfFrame;
    Timeline.finishSync();
  });
}
```

在这里，scheduleWarmUpFrame 不等待 Vsync 信号进而触发 window#draw 回调，而是直接自行发现，进行一次绘制，并且使用锁定了事件分发，保证第一次绘制完成前不会有别的绘制插件来（dart 的消息循环机制）。

handleBeginFrame 和 handleDrawFrame 在上面篇章 SchedulerBinding 有讲到，都是执行绘制任务，这些任务在 WidgetsBinding、各种Widget 中均可找到注册。

渲染树的绘制在上面讲到的 RendererBinding 中具体体现。

```dart
/// Pump the rendering pipeline to generate a frame.
///
/// This method is called by [handleDrawFrame], which itself is called
/// automatically by the engine when it is time to lay out and paint a frame.
///
/// Each frame consists of the following phases:
///
/// 1. The animation phase: The [handleBeginFrame] method, which is registered
/// with [Window.onBeginFrame], invokes all the transient frame callbacks
/// registered with [scheduleFrameCallback], in registration order. This
/// includes all the [Ticker] instances that are driving [AnimationController]
/// objects, which means all of the active [Animation] objects tick at this
/// point.
///
/// 2. Microtasks: After [handleBeginFrame] returns, any microtasks that got
/// scheduled by transient frame callbacks get to run. This typically includes
/// callbacks for futures from [Ticker]s and [AnimationController]s that
/// completed this frame.
///
/// After [handleBeginFrame], [handleDrawFrame], which is registered with
/// [Window.onDrawFrame], is called, which invokes all the persistent frame
/// callbacks, of which the most notable is this method, [drawFrame], which
/// proceeds as follows:
///
/// 3. The layout phase: All the dirty [RenderObject]s in the system are laid
/// out (see [RenderObject.performLayout]). See [RenderObject.markNeedsLayout]
/// for further details on marking an object dirty for layout.
///
/// 4. The compositing bits phase: The compositing bits on any dirty
/// [RenderObject] objects are updated. See
/// [RenderObject.markNeedsCompositingBitsUpdate].
///
/// 5. The paint phase: All the dirty [RenderObject]s in the system are
/// repainted (see [RenderObject.paint]). This generates the [Layer] tree. See
/// [RenderObject.markNeedsPaint] for further details on marking an object
/// dirty for paint.
///
/// 6. The compositing phase: The layer tree is turned into a [Scene] and
/// sent to the GPU.
///
/// 7. The semantics phase: All the dirty [RenderObject]s in the system have
/// their semantics updated (see [RenderObject.semanticsAnnotator]). This
/// generates the [SemanticsNode] tree. See
/// [RenderObject.markNeedsSemanticsUpdate] for further details on marking an
/// object dirty for semantics.
///
/// For more details on steps 3-7, see [PipelineOwner].
///
/// 8. The finalization phase: After [drawFrame] returns, [handleDrawFrame]
/// then invokes post-frame callbacks (registered with [addPostFrameCallback]).
///
/// Some bindings (for example, the [WidgetsBinding]) add extra steps to this
/// list (for example, see [WidgetsBinding.drawFrame]).
//
// When editing the above, also update widgets/binding.dart's copy.
@protected
void drawFrame() {
  assert(renderView != null);
  pipelineOwner.flushLayout();
  pipelineOwner.flushCompositingBits();
  pipelineOwner.flushPaint();
  if (sendFramesToEngine) {
    renderView.compositeFrame(); // this sends the bits to the GPU
    pipelineOwner.flushSemantics(); // this also sends the semantics to the OS.
    _firstFrameSent = true;
  }
}
```

- flushLayout 布局
- flushCompositingBits 预处理，检查是否需要重绘
- flushPaint 绘制，只会绘制需要重绘的节点
- compositeFrame 生成绘制一帧的Scene对象，发送个GPU进行绘制

> drawFrame 对应的就是绘制流水线的具体操作
