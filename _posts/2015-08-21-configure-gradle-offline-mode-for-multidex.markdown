---
title: 配置支持multidex的gradle离线模式
layout: post
tags:
 - android
---

[TOC]

前言
---
前面关于dex65536的问题这边写过一篇[Blog](http://sanjayz.com/2014/09/29/subpackage-dex-&&-use-proguard.html).不过开发环境上的问题没有涉及，android由于单个classes.dex，中方法存储格式为short，不能超过65536个方法。开发环境环境通过修改参数dex.force.jumb=true，勉强算是跑起来了。不过随着项目的增大force.jumb也支持不了，这边比较诡异，一开始跑不起来，但是后面clean项目又跑得起来，直到项目持续增大，最终都跑不起来。这边有一种说法是dex里面有存string的一个变量也是short，也不能超过65536，但是force.jumb模式并没有处理这个。

目前公司打包机子上的ant编译脚本适配过分包，这边打出来的包没有问题。不过由于一些历史原因这个打包脚本本地并不可以直接使用，借着这个契机，打算提议切换ant到gradle编译环境，由于公司是内网开发，内网并不访问外网，就有了这篇配置本地gradel环境。

配置gradle离线模式
======
按照***android官方的分包[方案](https://developer.android.com/reference/android/support/multidex/MultiDex.html)***在网络环境OK的地方可以很方便的实现android分包（需要科学上网）。无奈公司内网不能访问网络，这边采用了gradle的离线模式，算是留个档，具体方案见下：


方案流程
---
###1. 配置gradle环境
我这边用的是2.4,这个版本说是对android编译速度有显著提高，当然你也可以用目前最新的[gradle2.6](https://gradle.org/).这边配置离线模式，直接使用的是联网机子上的成功运行的缓存文件。由于以前缓存的是2.4的版本，我这边就沿用了。

1.  解压gradle程序到目录，并配置环境变量，方便代码调用
2.  解压gradle缓存目录到用户下.gradle文件下。gradle的默认缓存目录是这个，放在其他目录并没有试过。

###2. 配置SDK环境
1.  SDK中需要有支持multidex的buildTools版本，没有的请更新
2.  SDK中需要有multidex代码库，没有的请更新android-support。这边要注意的是需要配置ANDROID_HOME环境变量，这样android-gradle才能找得到multidex的jar包

###3. 配置开发环境
1. intellij idea只要使用gradle项目导入，再配置gradle为offline-mode即可。另外这边按照电脑的配置[加速AndroidStudio/Gradle构建](http://blog.isming.me/2015/03/18/android-build-speed-up/)


更多
---
这边算是留个档吧，刚开始离线模式使用的那个gradle，直接配成了缓存目录的.gradle里面的版本，一直不行，后来是multidex包没找到。当然这边有网的话都不是事，残念的是有时并不能连上网，哈哈！

