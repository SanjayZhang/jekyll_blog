---
title: android 构建library项目
layout: post
tags:
 - android
---

[TOC]


前言
---
要提供sdk给第三方项目，考虑使用library项目。用的是intellij,测试时，将原先的项目改成了library项目，自建主项目调用activity，两个manifest用的是不同包名，library项目一直报找不到R错误。根据先前的经验，反射处理R文件,处理R资源switch case为ifelse，更改xmlns域，运行项目成功。跑成功之后，library项目直接使用R文件访问也成功了。。。。原先以为只有eclipse处理了，生成两个R资源，没想到intellij也是可以的，只是eclipse生成在同一目录，intellij默认是分别在各自的project中。至于为什么一开始将原先项目改成library项目，不能通过R资源访问，应该是原先是application项目缓存的问题吧。。。。。。
另外要说的是，不通过library项目直接打成jar的话，还是需要反射加载R资源，直接在主项目中调用，当然相应的资源还是要拷贝至主项目。

library项目构建
======

项目大了，或者外包需求，需要提供SDK给项目调用。这边处理的方式一开始的做法是，直接提供jar包，加上相应的资源文件拷贝至主项目。这种做法适合资源文件较少的情况，实际上构建项目就是按照一个项目构建。另外一种做法是提供library项目，这边构建项目就分成的主项目和lib项目，lib项目可以是多个也可以有依赖关系。由于公司使用的是自己重写的ant，并不支持这个lib项目的构建模式（原生的ant脚本，貌似也不支持，需要自己添加项目模块，大雾？）。果断gradle是亲儿子，但是公司也不是说换就换构建方式的，到时候如何要使用library项目，还是得，再重写ant脚本支持。 当然，如果不走自定义ant脚本，目前的idea都是支持library项目。



方案流程
---
###1. library项目
1. 构建library项目使用library模式
2. 主项目调用library项目
这边的library项目，可以是源码，也可以是jar包。

intellij 14中这边的src和res资源默认是可以直接被主项目引用的，而assets资源需要配置，配置后idea会自动拷贝至主项目的assets( ***Android facet:setting "include assets from dependencies into apk" ***). AndroidManifest.xml也需要配置(***Android facet:setting "enable manifest merging"***)，不然得自己拷贝合并。

要注意的是这边library项目，其中自定义的资源，在xml中引用的话需要将xmlns的命名空间要改为
```xmlns:xxx="http://schemas.android.com/apk/res-auto"```

由于library项目新生成的R资源不具有final属性，不能和原先一样当常量对待，这样代码中的switch case都需要更改为ifelse


如果使用jar包的方式提供代码，也可以使用library这种方式，不过这边的jar中使用到的R资源需要反射获取。其中styleable资源比较特殊，返回的有数组，需要特殊处理，具体可以看最后的附件。

另外由于原先项目工程较大，这边推荐正则表达式 替换R资源。要看正则的点[这里](http://deerchao.net/tutorials/regex/regex.htm).

###2. jar加拷贝资源
1. 项目代码生成jar
2.  拷贝资源到需要调用的项目

这种方式比较好理解，就直接是一个项目，按照原先的流程走就好了。不过要注意的是，由于主项目的包名不固定，这边的jar中使用到的R资源需要反射获取。接触的国内大多数提供第三方SDK的，都是这种模式。当然如果是不给第三方调用的jar，可以在生成jar时使用同一个包名，这样就不用处理R资源。

当然这只是个人的理解，官方的android.jar就是所有资源都在里面，而且直接将R.class直接放进去。估计这边的R资源生成的id是特殊处理，设定了个范围，平常项目中自动生成的id是不会和这个冲突。


更多
---
- 这边要说的是使用idea开发的情况，如果使用脚本使用lib项目的构建模式，原生的ant脚本，貌似不支持，需要自己添加项目模块（大雾？）。这个对官方提供的ant build.xml不了解。但是由于官方不维护了，能用gradle的，都用它吧，毕竟亲儿子。
- [自用反射获取的R文件](https://onedrive.live.com/redir?resid=e6010db69e736b71!142&authkey=!AM-pT3Lbyjg-GoE&ithint=file%2cjava)
