---
title: android重复资源清理
layout: post
tags:
 - android
---

[TOC]

前言
---
公司断网，blog也好久没更新了，主要还是没有发现有什么可以写、本想把前段时间找的材料书签整理下，发现已经要的资源了，还是整理材料的频率太低了。今天主要是android清理重复资源的问题，虽然最新的android Gradle插件已经支持清理重复文件了，但是由于公司的项目ant编译的，体验不了，还是得自己动手。由于原文找不到了，这边就没链接了，还请原作者见谅！我这边也算是留个档吧。这个的工作具体也都是第三方实现的，这边只是让它更易于使用，也只是提供下思路，程序基本都是有规律的，有规律就可以根据程序解决问题。

无用资源清理
======

借助eclipse插件ucdetector，配合脚本清除java中的无用代码。
借助android lint，配合脚本清楚res中无用的代码
PS：当然ucdetector和lint的功能远不止这两个功能，这边刚好需要这边的两个功能，在配合程序 字符串处理，实现半自动删除无用的代码

解决的问题
---
1.  清除src 无用java代码
2.  清除res 无用资源

问题产生的原因
---
程序版本迭代，总有些遗留问题，尤其是多人协作

  
方案流程
---
###1. 清除src 无用java代码
1. 通过eclipse配合插件ucdetector产生报告
2. 运行清除代码
这边主要根据报告获取字符截取，定位类的地址，再删除
核心代码如下：
```
if (line.contains("Class") && line.contains("has 0 reference") && !line.contains("Method")) {
	int end = line.indexOf(".<init>");
	if (end != -1) {
		sb.append(line.substring(0, end));
		sb.append("\r\n");
	}
}
```

###2. 清除res 无用资源
1. 通过android提供的lint工具产生报告
2. 运行清除代码
这边主要根据报告获取字符截取，定位类的地址，再删除
核心代码如下：
```
if (line.contains("UnusedResources") && !line.contains("res\\values") && !line.contains("appcompat")) {
	int end = line.indexOf(":");
	if (end != -1) {
		sb.append(line.substring(0, end));
		sb.append("\r\n");
	}
}
```
 
  
更多
---
1. ucdetector以及lint本身无法 清除的资源，这边当然也无能为力啦。这边只是站在巨人的肩膀上，再想省事点。
2.  当然据说新版Gradle插件已经可以通过lint自动清除无用的代码，由于使用的是ant，这边只能自己做，有条件的当然用他们的，毕竟他们会持续维护
