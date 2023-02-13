---
layout: post
title: 自定义 Glide Target 实现动态 TextView ImageSpan
date: 2021-04-15 16:42:34
updated: 2021-04-15 16:42:34
tags:
  - glide
  - custom target
  - textview
  - imagespan
categories: Android
---

😄 记录下实现下图效果的过程

![](1.jpg)

<!-- More -->

## 思考

原先这里只是一个`TextView`，并不需要显示图片，所以只需要将数据加入`/`间隔即可，然而现在需要每种属性展示对应的图片，并且该图片还需要支持`GIF`，本来想换成`ImageView+TextView列表来显示`，但需求还需要保持原先的一行限制，也就是文字过长时显示`...`，此时如果弃用单`TextView`的话还需要计算多个控件的宽度，所以还是继续使用`单TextView`方案方便，幸好`Span`可以很轻松的将文字替换成其它形式展示，所以实现起来应该很简单。

## 设计

整体思路就是下载图片得到`Drawable`，包装进`ImageSpan`，并将其添加到`TextView`的文本中

```kotlin
val text = SpannableString("  红色")
text.set(0..1, ImageSpan(drawable))
textView.text = text
```

## 问题

因为要支持`GIF`，所以下载下来的`Drawable`如果是`Animatable`，还需要进行`start()`调用进行播放，并刷新`TextView`。

有`start`自然要有`stop`，否则动图将会一直播放。

原先我只是通过`Glide`来下载图片拿到`Drawable`，如下代码所示，那我还需要处理好生命周期，在页面关闭或者其它时机关闭动画，虽然现在挺多方式可以处理，但这种方式总是要外部介入，我想提供一个工具来实现这个功能并内部接管好生命周期的处理要如何实现呢？

```kotlin
Glide.with(textView)
    .load(url)
    .asDrawable()
    .submit(SIZE, SIZE)
    .get()
```

## Glide Target

`Glide`内部帮我们处理了生命周期的回调，自动并及时关闭图片加载请求，查看源码不难发现`Glide.into(imageView)`最后都是包装成一个`Target`，`Target`实现了`LifecycleListener`，所以完全可以自定义一个`Target`，借助`Glide`内部的生命周期管理来实现该功能。

借鉴`Glide`的`ImageViewTarget`，看到其继承自`ViewTarget`，但弃用了，推荐使用`CustomViewTarget`，那就实现一个

```kotlin
class MyTarget : CustomViewTarget<TextView, Drawable>() {
    override fun onResourceReady(resource: Drawable) {
        // setSpan
    }
    override fun onResourceCleared() {
        // removeSpan
    }
    override fun onStart() {
        // animatable.start()
    }
    override fun onStop() {
        // animatable.stop()
    }
}
```

现在就可以直接像平时`Glide.into(imageView)`一样直接调用了

```kotlin
Glide.with(textView)
    .load(url)
    .into(MyTarget(textView))
```

## 问题又来了

回看效果图，当有多张图片时，自然需求加载多个图片，那就是

```kotlin
urls.forEach(url -> {
    Glide.with(textView)
        .load(url)
        .into(MyTarget(textView))
})
```

当此时会发现只有最后一张图显示得出来，为什么呢？

![](2.jpg)

原来`Glide.into`时，会将之前的请求给清除掉，而这个请求是绑定在`View`上的（下面源码），在这里就是绑定在了`TextView`上。

```java
# CustomViewTarget

@Override
public final void setRequest(@Nullable Request request) {
  setTag(request);
}
private void setTag(@Nullable Object tag) {
  view.setTag(VIEW_TAG_ID, tag);
}
```

`Glide`这里的设计是`一个View对应一个Request`，思考下也合理，`一个ImageView`也就是为了展示`一张图片`，但我这里其实是`一个TextView对应多个Request`，那自然想到重写`setRequest`和`getRequest`，但发现`CustomViewTarget`的这两个方法修饰成了`final`，那只能跟`CustomViewTarget`说再见，跟他的爸爸`Target`说哈喽。

```kotlin
// 将tag变成map，将request和url绑定
class MyTarget(textView: TextView, url: String) : Target<Drawable> {
    override fun setRequest(request: Request?) {
        textView.getTag(VIEW_TAG_ID)?.let {
            it as MutableMap<String, Request?>
            it[attr] = request
        } ?: run {
            textView.setTag(VIEW_TAG_ID, mutableMapOf(url to request))
        }
    }

    override fun getRequest(): Request? {
        textView.getTag(VIEW_TAG_ID)?.let {
            it as MutableMap<String, Request?>
            return it[url]
        }
        return null
    }
}
```

剩下的就是一些细节处理了，包括拿到`Drawable`后如何定位到具体位置、`Animatable`播放如何刷新`TextView`、`ImageSpan`居中问题、`setText`时`BufferType`细节、占位图实现等。

## 总结

至此效果成功实现，外部只是要一行代码调用即可，不需要担心内存泄漏。
