---
title: 在assembleRelease之前执行自定义任务
date: 2024-03-25 14:19:04
tags:
  - Gradle
  - Kotlin
  - Android
---

### 背景

在实际的Gradle项目开发中，我们总是会遇到一些需求，要在release编译的时候执行一些任务，但debug时不需要。然而，Gradle编译有自己的一套生命周期，比如Android项目的assembleRelease任务在编译启动之前是没有办法静态获取到的。

下面我们就以“去除release版本中的logcat日志打印”为例，做一个简单的梳理。

### 源码

修改模板（module）级别的 `build.gradle.kts` 文件，我们初步目标是在编译刚启动但还没实际开始执行任务时插入我们的自定义任务，切入点就是Gradle的preBuild任务，这个是预定义的，所以可以进行静态配置：

```kotlin
plugins {
    id("com.android.application")
    id("kotlin-android")
    id("kotlin-kapt")
}

android {
    namespace = "com.xxx.yyy"
    compileSdk = 34

    // ...
}

dependencies {
    // ...
}

// 重点部分
tasks {
    preBuild {
        // 编译启动后，相关任务会动态生成，于是可以获取到
        val isAssembleRelease = gradle.startParameter.taskNames.any {
            it.contains("assembleRelease")
        }
        if (isAssembleRelease) {
            // 做一些release编译才做的事
        }
    }
}
```

以上可以说就是一个基础模板代码。此外，我们还可以添加本地配置来控制编译任务的开关，在项目（project）级别目录下添加 `local.properties` 文件，注意这个文件不要添加到git索引中，只是一个本地文件，一般来说项目拉取到本地的时候IDE（如Android Studio）会自动生成此文件：

```
## This file must *NOT* be checked into Version Control Systems,
# as it contains information specific to your local configuration.
#
# Location of the SDK. This is only used by Gradle.
# For customization when using a Version Control System, please read the
# header note.
#Wed Oct 25 11:29:50 CST 2023
sdk.dir=/Users/xxx/Library/Android/sdk

# 增加一个自定义的属性，等会儿使用
BUILD_FOR_RELEASE=true
```

回去修改模板代码，这样就可以在不需要的时候，关闭自定义任务的执行：

```kotlin
// kts文件头部需要import此类
import java.util.Properties

// ...

tasks {
    preBuild {
        val isAssembleRelease = gradle.startParameter.taskNames.any {
            // 把app bundle编译也考虑进去
            it.contains("bundleRelease") || it.contains("assembleRelease")
        }
        val localPropsFile = File(project.rootDir, "local.properties")
        val localProps = localPropsFile.takeIf { it.exists() }?.let {
            Properties().apply { localPropsFile.inputStream().use { input -> load(input) } }
        }
        val toggle = localProps?.getProperty("BUILD_FOR_RELEASE")?.toBoolean() ?: true
        if (isAssembleRelease && toggle) {
            // 做一些release编译才做的事
        }
    }
}
```

接下来，我们开始自定义task，去除release版本中的logcat日志打印：

```kotlin
tasks {
    val removeLogs by registering {
        group = "你可以随意定义分组名字，方便在Gradle任务列表中快速查找"
        doLast {
            val srcDir = File("src/main/java")
            srcDir.walkTopDown().filter {
                it.isFile && it.extension == "kt"
            }.forEach {
                val content = it.readText()
                it.writeText(
                    content.replace("import android.util.Log", "")
                        .replace("Log.", "// Log.")
                )
            }
        }
    }
}
```

这个任务只是个例子，很简单，目的就是搜寻源码目录下所有的kt文件，然后把logcat相关的代码注释掉。这样可以有效防止被反编译后泄露敏感字符串，当然你可以根据自己的需求随意去修改文件内容。

自定义之后，就是最后一步了，把任务插入添加到Gradle的编译任务链中，主要使用 `dependsOn` 方法，让preBuild任务依赖于我们的自定义任务：

```kotlin
tasks {
    val removeLogs by registering {
        // ...
    }

    preBuild {
        // ...
        if (isAssembleRelease && toggle) {
            dependsOn("removeLogs")
        }
    }
}
```

`dependsOn` 方法还接受多个参数，比如上述若改成 `dependsOn("task1", "task2", "task3")`，那么在preBuild之前就将按序依次执行这3个task。 

记得进行Gradle Sync操作，再启动assembleRelease编译，就可以在构建输出中看到我们自定义的任务了：

```
Executing tasks: [:app:assembleRelease] in project /Users/xxx/xxx/xxx

Starting Gradle Daemon...
Gradle Daemon started in 1 s 244 ms

> Configure project :app

> Task :app:createReleaseVariantModel UP-TO-DATE
> Task :app:removeLogs
> Task :app:preBuild
> Task :app:preReleaseBuild
```

可以看到，createReleaseVariantModel这个预定义的任务其实就是在动态生成assembleRelease，然后removeLogs在preBuild的前面，这样就能保证在代码优化和混淆开始之前，对源码进行自定义修改。

### 补充

对于大多数项目来说，都是有git管理的，所以在我们刚才对源码进行修改之后，其实并不想提交这些修改，而是想在编译完成后撤销修改。那么，我们就可以继续优化上述代码，在assembleRelease完成之后执行简单的git命令：

```kotlin
tasks {
    val removeLogs by registering {
        // ...
    }

    // 我们可以写一个方法，方便复用
    fun execGitRollbackAndCleanSrc() {
        exec { commandLine("git", "checkout", "src") }
        exec { commandLine("git", "clean", "-fd", "src") }
    }
    
    preBuild {
        // ...
        if (isAssembleRelease && toggle) {
            dependsOn("removeLogs")
            doLast {
                findByName("bundleRelease")?.doLast { execGitRollbackAndCleanSrc() }
                findByName("assembleRelease")?.doLast { execGitRollbackAndCleanSrc() }
            }
        }
    }
}
```

和 `dependsOn` 类似，`doLast` 也是调整任务链的有效方法之一，即在preBuild执行完成时，动态地给assembleRelease添加一个任务，使得assembleRelease在执行完成时，再执行我们的方法。

### 后话

上述代码都是一些简单的Gradle脚本，专用于自己项目的一些编译优化，进行大量实践后，完全可以开发成Gradle插件来使用，关键代码都是差不多的，需要的只是熟读Gradle官方文档，了解整个编译构建流程。
