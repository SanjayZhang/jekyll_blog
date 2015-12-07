---
title: 配置build.gradle以支持eclipse版本的多项目开发
layout: post
tags:
 - android
---

[TOC]

前言
---
前面ant转gradle写过一篇[Blog](http://sanjayz.com/2013/11/15/inti-gradle.html)。发展到现在，很多原先的需要自己写的脚本，目前官方已经加了相关参数。由于公司多版本项目的构建方式，沿用了ant编译时目录结构，需要适配相应的功能，这边也算是留个档吧


配置build.gradle以支持eclipse版本的多项目开发
======
目前的开发模式，开发的时候有个主版本，所有功能均在主版本开发。然后有个项目的配置文件，区分各个项目不同的状态。

具体自动打包的时候，由一台linux机子操作，先将主版本的代码从代码库中下下来，然后下配置文件。下完配置文件后，将具体的资源文件类似图片布局，通过直接替换，将配置文件中的内容换到主项目中；资源ID通过合并的方式，配置文件中和主版本重复的资源用配置文件中的相关代替。最后是处理程序packagename，将需要更改部分修改掉。

但是到开发的时候， 由于配置的一般是win的机子，相关由于目前自动打包的脚本没有win版本，需要手动把配置文件拷贝到主项目文件，然后手动进行重复资源id的合并（当然一般也不做，一般都是String资源，只是显示的区别），比较繁琐。

实际操作过程中，基本一个项目是由同一个开发管理，配置文件也只需要拷贝一次即可，但作为一个开发，这边工作显然是由脚本来完成比较geek。由于谷歌官方支持了[多项目开发](https://developer.android.com/tools/building/plugin-for-gradle.html)的管理，就打算使用官方的方案。这边要解决的问题是，原先的项目结构是eclipse的ant编译版本，需要进行gradle版本适配。其他也不说了，具体看下面的build.gradle
![build.gradle_version_variants_1](/media/files/2015/12/07/build_gradle_version_variants_1.png)
![build.gradle_version_variants_2](/media/files/2015/12/07/build_gradle_version_variants_2.png)
![build.gradle_version_variants_3](/media/files/2015/12/07/build_gradle_version_variants_3.png)


更多
---
1. 由于公司内网不能访问外网，这个脚本的gradle插件还是用的老的，目前官方已经升级至1.5，全面支持[databinding](https://developer.android.com/tools/revisions/gradle-plugin.html)
2. 这边的脚本名称是是叫多项目，实际上开发过程是单项目开发，按照这种方式只是便于管理
3. 另外这边说个小技巧，android这边的gradle，命令支持驼峰简称。例：gradle installDebug 直接输 gradle iD即可。当然前提是缩写之后不能有重复的命令

