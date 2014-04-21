---
title: Trim BOM Of UTF-8
layout: post
tags:
  - android
---

new JSONObject一直报错，传过来的数据打出来头部有空，以为是空格，trim（）不掉。后发现是BOM头，奇葩的是，两个String一个传过来头部是一个空，另一个是两个空，而第一个可以new JSONObject对象，第二个确不行。。。看源代码发现new 对象的时候只会去一次BOM头，改了下代码，去掉所有BOM头：

```
   public static String trimBomUTF8(String in) {
        while (in != null && in.startsWith("\ufeff")) {
            in = in.substring(1);
        }
        return in;
    }
```
  

PS:BOM是“Byte Order Mark”的缩写，用于标记文件的编码。

PS2:php端说是同样的代码，为啥么会产生2个BOM头。。。改动代码其实就是把if换成while，封装的时候它们没想到？应该不是吧，或许奇葩的问题应该是为啥会有两个BOM头，233。。。

PS3:new JSONObject原来本身也是不去头的，后来改的。改后就不需要去头，没遇到有两个头的问题，就不会发现还有BOM头这回事。问题是一直存在的，只是没有发现而已。。。
