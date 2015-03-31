---

title: dex分包（续）
layout: post
tags:
 - android

---

[TOC]

  
------

***目前官方已经支持分包，具体查看[官网](https://developer.android.com/reference/android/support/multidex/MultiDex.html)***

------

Dex分包方案  
===
Dex分包方案流程：dex分包，dex注入。对于android2.3之前的，还需要hack davlik虚拟机。  
上一篇没有加混淆，这次混淆研究的老半天。当然还研究了dex的分包原理，应用于公司项目。不过还是官方的build脚本牛逼，还可以研究研究。下面是算是干货吧，与上一篇有重复的地方。

解决的问题
---
1. android2.3对于单个classes.dex不能超过65536个方法，不然不能安装。（android2.3之后，通过修改参数dex.force.jumb=true）  
2. android2.3动态加载dex，davlik虚拟机有内存限制5M,超限会force close。（android2.3之后这个内存为8-16M） 

问题产生的原因
---
1. 在应用的安装过程中，会运行一个程序（dexopt）去根据当前机型为应用做一些特定的准备和优化。dexopt中使用了一个固定大小的缓冲区（名为LinearAlloc）来存储应用中方法的信息。最新的几个Android版本中，该缓冲的大小是8MB或16MB。但是Froyo（2.2）和Gingerbread（2.3）只有5MB。因为老版Android的这个限制，当方法数量超过这个缓冲大小的时候，会导致应用崩溃。
2. dexopt使用LinearAlloc储存dex文件的方法信息的时候，运行期的Android应用使用它访问实际用到的方法。如果在运行期将所有的dex文件都加载到一个进程中的话，实际的方法数量还是会超过限制。应用虽然可以启动但是很快就会崩溃。

***感谢作者分享[Facebook的Dalvik运行期补丁](http://log4think.com/facebook_dalvik_patch_for_android/)***
  
Dex分包方案流程
---
###1. 分包的方式
dex由主程序中的类以及使用的jar包中类共同生成。程序中的类通过javac生成class，jar包相当于class类的zip合集不用处理。打包的时候将两者共同打入dex文件。要分包原则上要计算方法的个数，等超过65536就打入第二个dex。当然由于功能模块的划分，我这里把主程序中的类打入原本的classes.dex,把第三方jar包打入第二个dex、第二个dex包存放在asset下面，命名为libs.apk。

这里要说明的是dex，apk，经过dx之后的jar包对于Android主程序来说是一致的都可以通过DexClassLoader加载 。***不熟悉ant打包android apk流程的，可以猛戳[这里](http://blog.csdn.net/chenzhiqin20/article/details/8191889)***
  
分包过程中涉及的流程是：生成dex文件，原来这里是打包经过混淆之后class文件以及jar包文件。这里的机制是前面生成的class文件混淆成jar包之后再解压出新的class文件（实际上jar包可以直接dx成dex，这里主要是可以再次分classes），再通过dx命令和jar包一起打成dex文件。

而分包方案中，这边只打包class文件，jar包在之前通过pre-dex打包成libs.apk放入asset目录。这里为了方便，才分class文件以及jar包文件，实际上可以，按照你的想法分。

另外要说的是这边的jar包都没参加混淆，混淆的是生成的class文件。如果需要混淆jar包，那这里的机制就要改，混淆之后的jar包，包含主程序与jar包的类，容易超出65536的限制，需要在混淆之后再分class，和分配原先不混淆的jar包，再生成两个dex。


###2. 主程序对第二个dex注入
主程序可以通过反射动态加载dex，但是有缺点：

* 第一，代码编写不方便，第二个类中的所有资源都要反射动态加载。
* 第二，如果分包的时候有一些android framework打入第二个包，主程序会启动不起来（网上的说法，有疑问，一般核心类不是应该存在系统里面的？)。

而dex注入是在activity启动之前的application通过反射替换该程序的内存中存类方法的地址。[这篇](http://blog.csdn.net/huli870715/article/details/38023065)的原理比较清晰，可以一看，不过没有关键代码。具体实现的时候没有成功估计是我自己实现的有问题，反射这类的又不好调试。这边的原理是将新增加的方法，通过反射动态添加到程序的PathClassLoader中的dexElements中。

而下面的方法直接将加载的dex替换掉程序PathClassLoader的父Classloader，原理当然还是需要跟源代码，但表示系统加载流程什么的还是交给[老罗的Android之旅](http://blog.csdn.net/luoshengyang/article/details/8923485)。目前猜测是程序调用方案先从PathClassloader取，若取不到则从他的父Classloader中取。

***感谢分享者[mmin18](https://github.com/mmin18/Dex65536)，以下是具体代码：***

```
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
  
###3. android2.3 davlik虚拟机内存限制5M提升到8M补丁  
这里主要用了[viila](https://github.com/viilaismonster/LinearAllocFix)的LinearAllocFix，将LinearAlloc扩大至8M。要说明的是路径info/viila/android/linearallocfix不能变，改的话需要jni重新生成so文件。  
  
限制&缺点
---
1. 二个dex包，方法数均不能超过65536  
2. 第二个dex包，不宜过大，不然程序启动会很慢。

改进&扩展 
---
1. 验证单个dex超标，或者直接根据方案数分包。  
2. 没有混淆第三方jar包，一般第三方都混淆了，当然总有意外需求。不过最重要的是没混淆就没删除没用的类   
3. 这个方案只是将dex分成两个包，当然还可以分更多   
4. 非主程序的dex包，还可以使用可逆加密算法，从程序中解密，提高程序的保密性。  
