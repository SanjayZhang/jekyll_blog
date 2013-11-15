---
title: 使用Gradle构建Android项目
layout: post
tags:
  - android
---
  
琢磨了两天Gradle,终于可以编译android项目了。记录留个档。  


过程中出现的问题：  

- 1:Android Libraries and Multi-project的设置  
- 2:打包后缺少*.so文件
- 3:多版本设置包名的问题  
- 4:ProGuard混淆出现无法编译错误  
- 5:多渠道打包问题  


主要参考的是[用Gradle 构建你的android程序](http://www.cnblogs.com/youxilua/archive/2013/05/20/3087935.html)  

- 问题1:参见[用Gradle 构建你的android程序-依赖管理篇](http://www.cnblogs.com/youxilua/archive/2013/05/22/3092657.html)
官方指南[Gradle Plugin User Guide](http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Goals-of-the-new-Build-System)  
- 问题2:参见[Gradle 构建 android 应用常见问题解决指南](http://www.cnblogs.com/youxilua/p/3348162.html)  
- 问题3:一直编译不通过，后来发现是资源文件中以包名路径调用了资源，配置文件里构建多版本时就不能改包名  
- 问题4:这个问题搞了很久，一直报task无法调用ProGuard.proguard。但是intellij是可以混淆的，后来发现是构建多项目时由于有两个v4包，为了省事使用
```
dependencies {
    compile fileTree(dir: 'libs', include: '*.jar')
}
```直接删掉了主项目中的v4包，但是混淆的配置文件中使用的是原来的，还是加载了项目中的v4包，但是gradle编译出现的错误看不出啥么花头。。。  
- 问题5:多渠道代码copy的是[Jenkins+Gradle实现android开发持续集成、打包](http://my.oschina.net/uboluo/blog/157483)  



附：build.gradle  

```
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:0.6.3'
    }
}

apply plugin: 'android'

dependencies {
    compile fileTree(dir: 'libs', include: '*.jar')
}

android {
    compileOptions.encoding = "UTF-8"

    compileSdkVersion 17
    buildToolsVersion "17"

    defaultConfig {
        minSdkVersion 8
        targetSdkVersion 17
    }

    dependencies {
        compile project(":libraries:xxx")
    }

    //签名
    signingConfigs {
        myConfig {
            storeFile file("xxx.keystore")
            storePassword "xxx"
            keyAlias "xxx"
            keyPassword "xxx"
        }
    }

    buildTypes{
        release {
            signingConfig signingConfigs.myConfig
            runProguard true
            proguardFile 'proguard-android.txt'
        }
    }

    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            resources.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
        }
    }

    productFlavors {
        playstore {

        }
        amazonappstore {

        }
    }
}
 //添加本地so库
task copyNativeLibs(type: Copy) {
    from(new File('libs')) { include '**/*.so' }
    into new File(buildDir, 'native-libs')
}

tasks.withType(Compile) { compileTask -> compileTask.dependsOn copyNativeLibs }

clean.dependsOn 'cleanCopyNativeLibs'

tasks.withType(com.android.build.gradle.tasks.PackageApplication) { pkgTask ->
    pkgTask.jniDir new File(buildDir, 'native-libs')
}

//替换AndroidManifest.xml的UMENG_CHANNEL_VALUE字符串为渠道名称
android.applicationVariants.all{ variant -> 
    println "${variant.productFlavors[0].name}"
    variant.processManifest.doLast{
        copy{
            from("${buildDir}/manifests"){
                include "${variant.dirName}/AndroidManifest.xml"
            }
            into("${buildDir}/manifests/$variant.name")

            filter{
                String line -> line.replaceAll("UMENG_CHANNEL_VALUE", "${variant.productFlavors[0].name}")
            }

            variant.processResources.manifestFile = file("${buildDir}/manifests/${variant.name}/${variant.dirName}/AndroidManifest.xml")
        }    
   }
}
```
