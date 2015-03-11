---
title: strange problem（java.io.EOFException ） encountered when using ksoap2-android-assembly-3.4.0-jar-with-dependencies.jar
layout: post
tags:
 - android
---


公司android项目用到了webservice，用的是ksoap2-android-assembly-34.0-jar-with-dependencies.jar包，期间出现了诡异的问题：手机放置一段时间后再开启，HttpTransportSE的第一次请求就会出错，报java.io.EOFException，然后稍后连续的请求一直是好的。但是放一段时间之后会重复出现该问题。没有具体了解机制，从解决方案来看，应该是http重连问题。

这边是***[stackoverflow中的解决方案](http://stackoverflow.com/questions/22680533/java-io-eofexception-using-ksoap2-lib-libcore-io-streams-readasciilinestreams-j)***，算是留个档吧。
