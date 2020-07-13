---
layout: post
title: 差分包
date: 2020-07-12 16:23:05
updated: 2020-07-12 16:23:05
tags:
  - apk
  - diff
  - patch
  - zip
categories: Android
---

调研Android差分包过程记录

<!-- More -->

## 前测

目前市场差分方案有
- bsdiff - 最常见的差分方案，例如国内应用市场的增量更新
- archive-patcher - 谷歌推出的基于bsdiff的优化方案，使差分包更小
- apkdiff - Github开源的项目，基于archive-patcher思想
- xdelta - 市面使用较少，但比bsdiff性能佳

以两个大约38M的apk作为例子
- bsdiff：差分包10M
- archive-patcher：得基于两个zlib包，所以得先将两个apk进行zlib压缩（体积可能会变大），但是差分效果佳，差分包400k
- apkdiff：使用lzma压缩，差分包300k，但是会导致合成包md5不一致，也可以先将apk进行zlib后再使用apkdiff，则md5相同
- xdelta：差分包10M，但是差分和合成的速度和内存优于bsdiff

## 分析

### bsdiff

- 比较常见的差异算法，不过因为较通用，所以没有考虑apk实际上是一个zip压缩包的情况，直接将apk当成文件去进行差异对比，所以产生的差异包较大
- patch时需要用到O(m+n)的内存

### archive-patcher

- 严格基于zlib，apk需要使用zlib重新压缩
- 差分和合成依旧使用bsdiff
- 之所以差分包大小比bsdiff小是因为先解压后再diff，比直接diff更容易描述差异
- 如果apk没有zlib，将直接bsdiff（差异包大小会与bsdiff一样大）
- 将apk解压后进行file-to-file的差异对比，得到差异包
- patch阶段将差异包压缩回apk（使用了zlib）得到最终apk包

### apkdiff

- 合成后只保证逻辑相同，不保证二进制相同，这一点可以通过zlib压缩后解决（因为解压和压缩用了相同的编码）
- 差分和合成使用hdiff，性能优于bsdiff
- 核心原理是将apk解压后进行差异对比，改动越小，差异包越小，比较符合增量更新意义
- 比archive-patcher暴力的是diff阶段直接解压然后差异对比，patch阶段直接zlib压缩回apk包，所以会导致二进制不相同
- 所以提供了ApkNormalized预处理apk包
- 需要注意的是，ApkNormalized重新压缩apk包后导致的二进制不相同会影响v2和v3的签名（v1不影响是v1不严格要求），所以如果项目使用v2或v3的签名方式需要在ApkNormalized后进行重签名

### xdelta

- 性能比bsdiff好以及稳定，但差异包大小与bsdiff差不多甚至更大一些
- 没有继续研究

## 接入成本

- 以armeabi-v7a为例
- jni以so实际大小
- java以jar实际大小

||bsdiff|archive-patcher|apkdiff|xdelta|
|----|----|----|----|----|
|接入方式|jni|java|jni|jni|
|接入大小|322KB|190KB|137KB||

## diff对比

- 由于diff过程属于预处理，一般放在服务器，所以只做时间对比
- 设备信息如下
- - macOs Catalina 10.15.4
- - 3.1GHz i5 16GB

||bsdiff|archive-patcher|apkdiff|xdelta|
|----|----|----|----|----|
|差分包生成时间|42s左右|24s左右|10s左右|4s左右|

## patch对比

- 测试机信息如下
- - 华为LRA-AL00 Android9 API28
- - 运行内存 8.0GB
- - 屏幕 2400 x 1080
- - 手机存储 128 GB
- - CPU架构 arm64-v8a

||bsdiff|archive-patcher|apkdiff|xdelta|
|----|----|----|----|----|
|差分包大小|10.6M|448k|304k||
|cpu占用|17%-24%|17%-22%|17%-23%||
|内存占用|86M|80M|6M||
|合成时间|5104ms|3841ms|3140ms||

> 实际做了挺多次对比，结果大致相同，所以不放出

## patch时多渠道因素

### 问题

前因
- 差异包只会打一个，以官方渠道包的新旧两个apk进行差异对比得到的差异包
- 线上会存在多种渠道包，目前渠道包以两种形式存在
- 1、ng-plugin生成的渠道包（在Zip Comment Central Directory区域实现）
- 2、Android Manifest中meta-data（部分渠道包还会更改应用名）

后果
- 由于diff&patch要求oldApk完全一致（渠道包因为Android Manifest文件不同已经破坏一致），线上用户使用本地渠道包进行patch违反这一条约，所以patch会失败

### 如何解决Zip Comment Central Directory

ng-plugin方案是在Central Directory的注释部分插入渠道信息，不会到apk造成破坏，可以正常安装，但注释也是数据的一部分，也会参与diff，所以会造成渠道包和原始包不一致导致无法patch

- 通过先抹除渠道信息再patch
- 如果需要保留原渠道信息，还可以再进行渠道的插入

### 如何解决Android Manifest

思路1
- 用户在拿到patch包后
- 利用本地渠道包还原oldApk，再以oldApk去进行patch得到newApk
- 难点1、还原oldApk过程需要与Jenkins打包流程一致才能保证100%还原
- 不足点1、还原所需时间=打包所需时间，用户端消耗不起

思路2
- diff阶段排除Android Manifest文件的差异对比
- 下发patch包同时下发新的Android Manifest文件
- patch阶段将新Android Manifest文件copy进去后再压缩回apk文件
- 难点1、保证生成的newApk与正常流程的newApk一致

## Demo

[https://github.com/izyhang/DiffPatchRecord](https://github.com/izyhang/DiffPatchRecord)