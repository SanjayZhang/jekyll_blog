---

title: Dex 分包（续）
layout: post
tags:
 - android

---

[TOC]

------

***目前官方已经支持分包，具体查看 [官网](https://developer.android.com/reference/android/support/multidex/MultiDex.html)***

------

## Dex 分包方案

Dex 分包方案流程：dex 分包，dex 注入。对于 Android 2.3 之前的，还需要 hack davlik 虚拟机。
上一篇没有加混淆，这次混淆研究的老半天。当然还研究了 dex 的分包原理，应用于公司项目。不过还是官方的 build 脚本牛逼，还可以研究研究。下面是算是干货吧，与上一篇有重复的地方。

## 解决的问题

1. Android 2.3 对于单个 classes.dex 不能超过 65536 个方法，不然不能安装。（Android 2.3 之后，通过修改参数 dex.force.jumb = true）
2. Android 2.3 动态加载 dex，davlik 虚拟机有内存限制 5M，超限会 force close。（Android 2.3 之后这个内存为 8-16M）

## 问题产生的原因

1. 在应用的安装过程中，会运行一个程序（Dexopt）去根据当前机型为应用做一些特定的准备和优化。Dexopt 中使用了一个固定大小的缓冲区（名为 LinearAlloc）来存储应用中方法的信息。最新的几个 Android 版本中，该缓冲的大小是 8MB 或 16MB。但是 Froyo（2.2）和 Gingerbread（2.3）只有 5MB。因为老版 Android 的这个限制，当方法数量超过这个缓冲大小的时候，会导致应用崩溃。
2. Dexopt 使用 LinearAlloc 储存 dex 文件的方法信息的时候，运行期的 Android 应用使用它访问实际用到的方法。如果在运行期将所有的 dex 文件都加载到一个进程中的话，实际的方法数量还是会超过限制。应用虽然可以启动但是很快就会崩溃。

***感谢作者分享 [Facebook 的 Dalvik 运行期补丁](http://log4think.com/facebook_dalvik_patch_for_android/)***
  
## Dex 分包方案流程

### 1. 分包的方式

Dex 由主程序中的类以及使用的 jar 包中类共同生成。程序中的类通过 javac 生成 class，jar 包相当于 class 类的 zip 合集不用处理。打包的时候将两者共同打入 dex 文件。要分包原则上要计算方法的个数，等超过 65536 就打入第二个 dex。当然由于功能模块的划分，我这里把主程序中的类打入原本的 classes.dex，把第三方 jar 包打入第二个 dex、第二个 dex 包存放在 asset 下面，命名为 libs.apk。

这里要说明的是 dex，apk，经过 dx 之后的 jar 包对于 Android 主程序来说是一致的都可以通过 DexClassLoader 加载。***不熟悉 Ant 打包 Android apk 流程的，可以猛戳 [这里](http://blog.csdn.net/chenzhiqin20/article/details/8191889)***
  
分包过程中涉及的流程是：生成 dex 文件，原来这里是打包经过混淆之后 class 文件以及 jar 包文件。这里的机制是前面生成的 class 文件混淆成 jar 包之后再解压出新的 class 文件（实际上 jar 包可以直接 dx 成 dex，这里主要是可以再次分 classes），再通过 dx 命令和 jar 包一起打成 dex 文件。

而分包方案中，这边只打包 class 文件，jar 包在之前通过 pre-dex 打包成 libs.apk 放入 asset 目录。这里为了方便才分 class 文件以及 jar 包文件，实际上可以按照你的想法分。

另外要说的是这边的 jar 包都没参加混淆，混淆的是生成的 class 文件。如果需要混淆 jar 包，那这里的机制就要改，混淆之后的 jar 包，包含主程序与 jar 包的类，容易超出 65536 的限制，需要在混淆之后再分 class，和分配原先不混淆的 jar 包，再生成两个 dex。

### 2. 主程序对第二个 dex 注入

主程序可以通过反射动态加载 dex，但是有缺点：

1. 代码编写不方便，第二个类中的所有资源都要反射动态加载。
2. 如果分包的时候有一些 Android Framework 打入第二个包，主程序会启动不起来（网上的说法，有疑问，一般核心类不是应该存在系统里面的？）。

而 dex 注入是在 Activity 启动之前的 Application 通过反射替换该程序的内存中存类方法的地址。[这篇](http://blog.csdn.net/huli870715/article/details/38023065) 的原理比较清晰，可以一看，不过没有关键代码。具体实现的时候没有成功估计是我自己实现的有问题，反射这类的又不好调试。这边的原理是将新增加的方法，通过反射动态添加到程序的 PathClassLoader 中的 dexElements 中。

而下面的方法直接将加载的 dex 替换掉程序 PathClassLoader 的父 Classloader，原理当然还是需要跟源代码，但表示系统加载流程什么的还是交给 [老罗的 Android 之旅](http://blog.csdn.net/luoshengyang/article/details/8923485)。目前猜测是程序调用方案先从 PathClassloader 取，若取不到则从他的父 Classloader 中取。

***感谢分享者 [mmin18](https://github.com/mmin18/Dex65536)，以下是具体代码：***

``` java
/**
 * Copy the following code and call dexTool() after super.onCreate() in
 * Application.onCreate()
 * <p>
 * This method hacks the default PathClassLoader and load the secondary dex
 * file as it's parent.
 */
@SuppressLint("NewApi")
private void dexTool() {
    File dexDir = new File(getFilesDir(), "dlibs");
    dexDir.mkdir();
    File dexFile = new File(dexDir, "libs.apk");
    File dexOpt = getCacheDir();
    try {
        InputStream ins = getAssets().open("libs.apk");
        if (dexFile.length() != ins.available()) {
            FileOutputStream fos = new FileOutputStream(dexFile);
            byte[] buf = new byte[4096];
            int l;
            while ((l = ins.read(buf)) != -1) {
                fos.write(buf, 0, l);
            }
            fos.close();
        }
        ins.close();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    ClassLoader cl = getClassLoader();
    ApplicationInfo ai = getApplicationInfo();
    String nativeLibraryDir = null;
    if (Build.VERSION.SDK_INT > 8) {
        nativeLibraryDir = ai.nativeLibraryDir;
    } else {
        nativeLibraryDir = "/data/data/" + ai.packageName + "/lib/";
    }
    DexClassLoader dcl = new DexClassLoader(dexFile.getAbsolutePath(),
            dexOpt.getAbsolutePath(), nativeLibraryDir, cl.getParent());
    try {
        Field f = ClassLoader.class.getDeclaredField("parent");
        f.setAccessible(true);
        f.set(cl, dcl);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```
  
### 3. Android 2.3 davlik 虚拟机内存限制 5M 提升到 8M 补丁

这里主要用了 [viila](https://github.com/viilaismonster/LinearAllocFix) 的 LinearAllocFix，将 LinearAlloc 扩大至 8M。要说明的是路径 info/viila/android/linearallocfix 不能变，改的话需要 jni 重新生成 so 文件。
  
## 限制&缺点

1. 二个 dex 包，方法数均不能超过 65536。
2. 第二个 dex 包，不宜过大，不然程序启动会很慢。

## 改进&扩展

1. 验证单个 dex 超标，或者直接根据方案数分包。
2. 没有混淆第三方 jar 包，一般第三方都混淆了，当然总有意外需求。不过最重要的是没混淆就没删除没用的类。
3. 这个方案只是将 dex 分成两个包，当然还可以分更多。
4. 非主程序的 dex 包，还可以使用可逆加密算法，从程序中解密，提高程序的保密性。
