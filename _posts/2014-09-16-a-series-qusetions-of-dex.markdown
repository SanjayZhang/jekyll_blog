---
title: A Serives Questions Of DEX
layout: post
tags:
 - android
---
  
7月底换工作以来一直没空写blog（主要是没东西写。。。）。最近碰到DEX超限问题，上个礼拜一开始弄，今天终于跑通了，当然后续还有东东要完善，这里标记下先。
  
引起的问题：Unable to execute dex: Cannot merge new index 65536 into a non-jumbo instruction!

dex中的方法数超过65536限制，在android2.3以后版本只要修改下编译的参数default.properties中```dex.force.jumb=true``` ,intellij的话选中Android Complier的Force jumbo mode；而如果是android2.3的机子，会悲催的装不上。这是由于android2.3机制中安装时单个dex文件不能超过65536。
  
解决这个问题，首先想到的是插件化开发，纠结的是动态加载的问题。发现动态加载方法倒没什么问题，但是项目中有activity打到jar里去，研究了一通发现反射运行activity还是有些麻烦。
网上动态加载未安装apk中的activity找到以下方案：
  
 - 不使用R资源动态生成布局，再预留个content入口，反射的时候直接根据这个context建立布局。这种方式在代码里不能使用this，R资源。
[探秘腾讯Android手机游戏平台之不安装游戏APK直接启动法](http://blog.zhourunsheng.com/2011/09/%E6%8E%A2%E7%A7%98%E8%85%BE%E8%AE%AFandroid%E6%89%8B%E6%9C%BA%E6%B8%B8%E6%88%8F%E5%B9%B3%E5%8F%B0%E4%B9%8B%E4%B8%8D%E5%AE%89%E8%A3%85%E6%B8%B8%E6%88%8Fapk%E7%9B%B4%E6%8E%A5%E5%90%AF%E5%8A%A8%E6%B3%95)
 - 实现一个大框架自己管理未安装的R资源，this等,[apkplug（采用OSGI技术）](http://www.apkplug.com/)。对于我等渣渣来讲，明显比前者更有吸引力。
  
但是，重要的是但是，由于项目的jar包不是自己开发，插件化调用的幻想破灭。再找方案的时候就发现[Android dex分包方案](blog.csdn.net/huli870715/article/details/38023065)。起先最大的问题是ant脚本的编写，主要还是对android打包机制的不熟。分完包发现，核心代码没有。。。嚓，对于对android存取函数方法完全不熟，真是无从下手啊。还是看看facebook英文原版吧，继而又发现[Facebook的Dalvik运行期补丁](http://log4think.com/facebook_dalvik_patch_for_android/)。思路豁然开朗，立马下facebook，反编译。哈哈，由于是反射调用，混淆的不彻底，依样画葫芦，补全博主的函数，一试，安装是安装了，但是运行跑不起了，我嚓不行，给跪了。还是研究下facebook的代码吧，你妹的核心代码反编译错误。诶，万念俱灰之时，终于找到了两盏明灯啊[mmin18（好像是大众点评的大神）](https://github.com/mmin18/Dex65536)，[Hack Dalvik VM解决Android 2.3 DEX/LinearAllocHdr超限](http://viila.info/2014/04/android-2-3-dex-max-function-problem/)。前者解决分包问题（自己分的包表示太渣了）以及dex注入，后者解决2.3超限。
  
程序算是跑起来了，不过原理也只了解了个皮毛，android dex优化的5M限制和65536到底有毛关系？dex注入这里有两种方案，先前用的不行，但是从facebook代码来看两种都有，还是有关版本（坑爹的反编译错误）？看来还是要研究源代码啊。采用的默认分包方案，混淆还需要自定义。采用的超限方案没有自己实现，只是调通了下大神的代码。。。
  
PS：吐槽下，ant签名时由于java版本升级，导致签名错误。。。还以为是签名不行，又自己生成了个--！当出现错误，总是把前面对的给改掉。。。一是，对于先前对的不确定。二是，错了自己阵脚就乱，算是老毛病了。当然这里是需要重新生成签名，2333！！！
  
  
其他参考的文献：
  
 - [Android动态加载jar/dex](http://www.cnblogs.com/over140/archive/2011/11/23/2259367.html)
 - [Android应用开发提高系列（4）——Android动态加载（上）——加载未安装APK中的类](http://www.cnblogs.com/over140/archive/2012/03/29/2423116.html)
 - [android 程序开发的插件化 模块化方法 之一](http://www.cnblogs.com/hangxin1940/archive/2011/12/14/2288169.html)
 - [under-the-hood-dalvik-patch-for-facebook-for-android](https://www.facebook.com/notes/facebook-engineering/under-the-hood-dalvik-patch-for-facebook-for-android/10151345597798920)
 - [custom-class-loading-in-dalvik](http://android-developers.blogspot.com/2011/07/custom-class-loading-in-dalvik.html)
 - [android调用第三方库——第一篇](http://blog.csdn.net/jiuyueguang/article/details/9447245)
 - [Android APK JNI sample (JAVA JNI)](http://blog.csdn.net/eqiang8271/article/details/9967511)
