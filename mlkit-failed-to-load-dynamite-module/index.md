---
layout: post
title: MlKitException Failed to load deprecated vision dynamite module
date: 2023-04-17 22:55:46
updated: 2023-04-17 22:55:46
tags: 
 - mlkit
 - barcode
 - google play services
categories: Android
---

记一次 *mlkit* 错误分析

<!-- more -->

## 前景

项目接入`mlkit`-`barcode`功能，考虑到国产大部分手机不支持`google play services`，遂使用模型捆绑包的依赖方式接入`sdk`

```groovy
implementation 'com.google.mlkit:barcode-scanning:17.1.0'
```

该引入方式会将`barcode`的识别模型一起打包到`apk`里，虽然包体积会大一些，但起码所有手机都能用

> 另一种[引入方式](https://developers.google.com/ml-kit/vision/barcode-scanning/android)需要依赖`google play services`，会动态下载识别模型

## 问题

在`demo`工程试接了下，国产手机运行正常，遂接入到项目工程，但运行报错

```
Invalid GmsCore APK, remote loading disabled.

Error preloading model resource
com.google.mlkit.common.MlKitException: Failed to load deprecated vision dynamite module.

Caused by: com.google.android.gms.dynamite.DynamiteModule$LoadingException: No acceptable module com.google.android.gms.vision.dynamite found. Local version is 0 and remote version is 0.
```

下载模型失败？我不是用的捆绑包吗？怎么还去下载模型？

## 分析

再重新看日志，发现没找到模型的版本都是*0*

```
Local version is 0 and remote version is 0.
```

在之前还有一句日志

```
DynamiteModule           W  Local module descriptor class for com.google.mlkit.dynamite.barcode not found.
```

找不到本地的模型，进`DynamiteModule`看看是怎么找的

```java
context = context.getApplicationContext();
ClassLoader context1 = context.getClassLoader();
var2 = new StringBuilder();
var2.append("com.google.android.gms.dynamite.descriptors.");
var2.append(moduleId);
var2.append(".ModuleDescriptor");
Class context2 = context1.loadClass(var2.toString());
var15 = context2.getDeclaredField("MODULE_ID");
context3 = context2.getDeclaredField("MODULE_VERSION");
var3 = Objects.equal(var15.get((Object)null), moduleId);
```

发现`DynamiteModule`会用反射去取`ModuleDescriptor`下的`MODULE_ID`和`MODULE_VERSION`两个常量值来获取模型版本

```java
@DynamiteApi
@RetainForClient
public class ModuleDescriptor {
    @RetainForClient
    @NonNull
    public static final String MODULE_ID = "com.google.android.gms.measurement.dynamite";
    @RetainForClient
    public static final int MODULE_VERSION = 70;

    public ModuleDescriptor() {
    }
}
```

找不到说明反射失败了，说明这个类没打包进`apk`或其它问题，反编译下`apk`发现两个常量没了，难怪找不到

## 原因

这么一想，项目接了个常量内联的插件，插件没检测出来误伤把这两个常量干掉了，导致加载本地模型失败

