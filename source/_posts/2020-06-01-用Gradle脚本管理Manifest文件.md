---
layout:     post
title:      用Gradle脚本管理Manifest文件
subtitle:   探索发现。
date:       2020-06-01
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - Gradle
    - Windows
---

## 编译时区分不同的manifest

很多Android项目都会区分debug和release的manifest文件，以便调试，一些组件化的项目甚至有多个manifest文件来调试不同的组件。举个简单的例子，在app的build.gradle文件中：

```groovy
android {
    defaultConfig {
        applicationId "com.xxx.xxx"
    }
    sourceSets {
        main {
            if(是否为debug打包) {
                manifest.srcFile "${projectDir}/src/main/debug/AndroidManifest.xml"
            } else {
                manifest.srcFile "${projectDir}/src/main/release/AndroidManifest.xml"
            }
        }
    }
}
```

这个地方的if条件，一般可以是一个变量，或者一个方法：

```groovy
// 这样
def isDebug = true
// 或者
def isDebug() {
    // 一些逻辑
    return true
}
// 以上均遵循Groovy语法规则，Android项目中的Gradle脚本都是用Groovy语言（一门JVM动态语言，很容易上手）编写的。
android { ... }
```

然后此处的 ${projectDir} 变量对应的是当前模块所在的路径，这里就是app的路径，不是整个工程的路径。这样一来，我们就能编译不同的manifest文件了。

## 自动判断编译类型

但是，这样每次编译都需要手动改那个debug变量，挺麻烦的，尤其是一些技术团队可能是用的公司服务器在线编译，每次都要提交代码到仓库。其实我们可以用gradle插件的特性，这样来判断：

```groovy
// 这里app是当前模块名，assembleDebug是gradle编译debug包时的默认task
if(gradle.startParameter.taskNames.contains(":app:assembleDebug")) {
    manifest.srcFile "${projectDir}/src/main/debug/AndroidManifest.xml"
} else {
    manifest.srcFile "${projectDir}/src/main/release/AndroidManifest.xml"
}
```

不用改代码，打release包和debug包就自动区分了，当然你也可以增加新的条件，去对应不同的编译task。

## 解析并自动生成manifest文件

这部分是本文重点啦！从上面的步骤来看，我们显然是准备了2份manifest文件，平时维护时也需要一起修改。实际业务中很可能debug和release的manifest内容差不多，可能只是某些组件节点的属性不同，手动改也挺麻烦的。

能不能通过gradle脚本动态地来修改并生成manifest文件呢？当然可以，本质上就是处理XML文件。AndroidManifest是标准的XML文件。正好，Groovy处理XML又非常的简单。这里我们还是以一个实际例子来讲。

比如我们的需求是，在debug测试时，应用正常显示桌面图标，在release发布时，应用需要隐藏图标。那么，两个manifest文件就是这样的：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest ...>
    <application ...>
        <!--App入口页面-->
        <activity
            ...
            android:name=".TestActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
...
```

release版本的manifest：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest ...>
    <application ...>
        <!--App入口页面-->
        <activity
            ...
            android:name=".TestActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
                <!--在入口Activity添加如下data节点即可隐藏桌面图标-->
                <data 
                    android:host="localhost" 
                    android:scheme="${applicationId}" />
            </intent-filter>
        </activity>
...
```

然后平时我们开发过程中，如果新增了一些四大组件，2个文件都要同时增加，而区别却只有入口Activity。我们想要的是release版本的manifest是自动生成的（在生成时插入那个data节点），平时只需要开发改动debug版本的文件即可。

主要思路比较简单：

1. 通过Groovy的XML解析库读取debug的manifest文件，遍历节点找到入口Activity；
2. 将data节点插入到入口Activity下面；
3. 把新的内容写入release版本的文件当中。

先直接看build.gradle脚本源码：

```groovy
import groovy.xml.XmlUtil

def getManifestPath(buildType) {
    return "${projectDir}/src/main/$buildType/AndroidManifest.xml"
}

def handleManifestXml(manifest) {
    if (gradle.startParameter.taskNames.contains(":app:assembleDebug")) {
        // debug build
        def debugPath = getManifestPath('debug')
        manifest.srcFile debugPath
        println("Manifest path: $debugPath")
    } else {
        // release build
        def releasePath = getManifestPath('release')
        println("Manifest path: $releasePath")
        manifest.srcFile releasePath

        def debugPath = getManifestPath('debug')
        def debugFile = new File(debugPath)
        def releaseFile = new File(releasePath)
        def debugXml = new XmlParser(false, false).parse(debugFile)
        // 修改debug的manifest，自动生成release版本
        debugXml.application[0].each { comp ->
            if (comp.name() == 'activity') {
                comp.each { filter ->
                    // 搜索入口Activity
                    if (filter.toString().contains('android.intent.category.LAUNCHER')) {
                        // 添加data节点，隐藏桌面icon
                        filter.appendNode('data', ['android:host': 'localhost', 'android:scheme': '${applicationId}'])
                        return true // break each闭包
                    }
                }
            }
        }
        releaseFile.write(XmlUtil.serialize(debugXml))
    }
}

android {
    defaultConfig {
        applicationId "com.xxx.xxx"
    }
    sourceSets {
        main {
            // 此处会自动处理manifest文件
            // 注意：平时开发时只需修改debug下的manifest即可，请勿手动修改release的
            handleManifestXml(manifest)
        }
    }
...
```

关键逻辑从 `def debugXml = new XmlParser(false, false).parse(debugFile)` 开始，这里的XmlParser构造方法可以不传任何参数，我这里传false主要是为了让manifest根节点自动添加 **xmlns** ，这样最后生成文件内容会简洁一点。

然后就是 `debugXml.application[0].each { comp -> ... } ` 循环块，Groovy的语法很神奇，这里可以直接通过节点名称来获取数组，比如application，取0当然就是第一个，一般我们的manifest里就一个application节点，所以也不会出现空指针异常。each是数组的函数，遍历数组的每一个节点，comp参数是我自定义的命名。能拿到application的子节点，后面就都是一些逻辑处理了，不赘述。

## 一些小问题

如果是Windows开发者，最后写文件时，需要注意换行符和编码的问题。

1、最后的write方法，加上参数：

```groovy
releaseFile.write(xmlStr, "UTF-8")
```

或者直接改全局配置，修改整个工程下的gradle.properties文件，在gradle的JVM参数后面追加如下：

```
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
```

2、换行符问题，没有JVM参数可供修改，只能手动处理，还是最后write时：

```groovy
releaseFile.write(XmlUtil.serialize(debugXml).replaceAll('\r\n', '\n'))
```

这里XmlUtil工具类的serialize方法返回类型实际上就是String，所以可以这样直接replace。
