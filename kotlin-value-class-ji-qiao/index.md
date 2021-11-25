---
layout: post
title: Kotlin Value Class 技巧
date: 2021-11-21 15:25:46
updated: 2021-11-21 15:25:46
tags:
  - kotlin
  - value class
  - inline
  - optimise
categories: Android
---

## inline-classes

[https://kotlinlang.org/docs/inline-classes.html](https://kotlinlang.org/docs/inline-classes.html)

## 如何使用

```kotlin
@JvmInline
value class Name(val s: String)

val john = Name("John")

fun say(name: Name) {}
```

## 思考

### 内联类是什么

1. 内联类必须明确声明一个参数，目的是为了运行时内联成该参数（在上述例子中 john 实际上就等于字符串 "John"）

### 为何要用内联类

1. 首先内联类是为了解决数据类型被包装后引起更多堆内存分配而带来的性能问题（内联后将少了一层包装）。
2. 其次内联类给了通用数据类型（Int、String）更多的含义（上述例子 john 虽然在运行时为普通的字符串，但在我们看来是一个有含义的类型 Name）。
3. 内联类可以获得更好的校验体验。（上述例子 say() 将只允许传 Name 类型而不是 String 类型）

### 什么时候用内联类

1. 用来代替@StringDef、@IntDef（而不是使用Enum）

以前Java可以用注解@IntDef来定义一些常量，编译器会自动校验参数合法性

```java
@Retention(SOURCE)
@IntDef({NAVIGATION_MODE_STANDARD, NAVIGATION_MODE_LIST, NAVIGATION_MODE_TABS})
public @interface NavigationMode {}
public static final int NAVIGATION_MODE_STANDARD = 0;
public static final int NAVIGATION_MODE_LIST = 1;
public static final int NAVIGATION_MODE_TABS = 2;
...
public abstract void setNavigationMode(@NavigationMode int mode);
@NavigationMode
public abstract int getNavigationMode();
```

但Kotlin中只能使用Enum来代替，这里其实可以用内联类来获取更少的内存占用

```kotlin
@JvmInline
value class NavigationMode(val value: Int)
val NAVIGATION_MODE_STANDARD = NavigationMode(0)
val NAVIGATION_MODE_LIST = NavigationMode(1)
val NAVIGATION_MODE_TABS = NavigationMode(2)
...
abstract fun setNavigationMode(mode: NavigationMode)
abstract fun getNavigationMode(): NavigationMode
```

2. 直接为数据类型定义特有功能（而不是使用扩展函数）

有时我们会为一些类扩展一些函数，比如为Long扩展一个formatDate成日期的函数

```kotlin
fun Long.formatDate(): String = SimpleDateFormat.getInstance().format(this)
```

当这样做导致所有Long类型都有这么一个扩展函数，当扩展函数定义得越来越多时，代码提示肯定会杂乱无章。此时就可以用内联类包装起来，为其定义他特有的功能

```kotlin
@JvmInline
value class Timestamp(val value: Long) {
  fun formatDate(): String = SimpleDateFormat.getInstance().format(value)
  ...
}
```

3. 还可以为内联类定义操作符方法，获得更好的类型校验

```kotlin
@JvmInline
value class UInt(val value: Int) {
  operator fun plus(other: UInt) = UInt(value + other.value)
  ...
}
```
