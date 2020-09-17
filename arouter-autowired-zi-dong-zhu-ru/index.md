---
layout: post
title: ARouter @Autowired 自动注入
date: 2020-05-31 11:30:08
updated: 2020-05-31 11:30:08
tags:
  - byte
  - inject
  - arouter
categories: Android
---

## 前言

ARouter 有一个@Autowired 的注解，能自动帮我们赋值一些变量，例如

<!-- More -->

```java
public class MainFragment {
    @Autowired
    String name;

    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Arouter.getInstance().inject(this);

        System.out.println(name);
    }
}
```

通过`Arouter.getInstance().inject(this);`就能将 Activity 传输的一些值自动帮我们赋值上对应变量，省去我们手动去调用`getIntent().getString(xxx)`

## inject(this)

那么看下它做了什么，翻了几下发现，ARouter 会做以下几步操作
1、APT 编译期间扫描@Autowired 字段的文件并生成 MainFragment$$ARouter$$Autowired 文件

- $$ARouter$$Autowired 类实现了 ISyringe 接口，拥有一个 inject(Object)的方法，这里面就是赋值的代码

```java
public class MainFragment$$ARouter$$Autowired implements ISyringe {
  private SerializationService serializationService;

  @Override
  public void inject(Object target) {
    MainFragment substitute = (MainFragment)target;
    substitute.name = substitute.getArguments().getString("name");
    substitute.age = substitute.getArguments().getInt("age");
  }
}
```

2、Arouter.getInstance().inject(this);则是通过反射创建了$$ARouter$$Autowired 类并调用 inject 方法实现自动赋值

```java
@Override
    public void autowire(Object instance) {
        String className = instance.getClass().getName();
        try {
            if (!blackList.contains(className)) {
                ISyringe autowiredHelper = classCache.get(className);
                if (null == autowiredHelper) {  // No cache.
                    autowiredHelper = (ISyringe) Class.forName(instance.getClass().getName() + SUFFIX_AUTOWIRED).getConstructor().newInstance();
                }
                autowiredHelper.inject(instance);
                classCache.put(className, autowiredHelper);
            }
        } catch (Exception ex) {
            blackList.add(className);    // This instance need not autowired.
        }
    }
```

## 思考

众所周知反射的性能是差的，那么有什么办法不反射吗，答案是有的，那就是直接
`new MainFragment$$ARouter$$Autowired().inject(this);`

然而在代码没编译之前，MainFragment$$ARouter$$Autowired 这个类是还没生成的，自然无法直接调用。况且这样还有个问题，那就是每个 Activity 或 Fragment 生成的类都是唯一的，我们也不可能在每个地方手动 new+inject，这样还不如反射来的方便

> 这时候就需要 Android 提供的工具 Transform 了

## Transform 开发

### 核心原理

> 利用 Transform，在编译期间往使用了@Autowired 的 Activity 或 Fragment 类的 onCreate(Bundle)方法自动注入`new xxx$$ARouter$$Autowired().inject(this);`这行代码

### 过程

1、扫描整个项目里名称后缀为$$ARouter$$Autowired 的 class 文件
2、以此找到对应的 Activity 或 Fragment 文件
3、利用 ASM 库对该文件进行访问
4、访问到 onCreate(Bundle)方法时，在 super.onCreate 前写入 inject 方法

### 结果

编译完成后，可以通过 apk 包分析或这在 app\build\intermediates\transforms 目录下找到 MainFragment 文件编译后的代码，可以看见 MainFragment 的 onCreate 方法里面多了一行代码，就是上面所想要的
`new MainFragment$$ARouter$$Autowired().inject(this);`

## 再次思考

 虽然利用 Transform 可以解决反射的问题，但无疑也带来了一个问题，就是项目协作上，其他人不了解的话会很奇怪。我的做法是在$$ARouter$$Autowired 类加了行注释，起码别人在看这个类的时候能知道什么时候会 inject。
至于怎么加这行注释，就靠各位发挥了

本项目例子已开源[Github](https://github.com/izyhang/ARouter-AutowiredTransform)
