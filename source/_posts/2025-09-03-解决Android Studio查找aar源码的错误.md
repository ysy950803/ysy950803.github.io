---
title: 解决Android Studio查找aar源码的错误
date: 2025-09-03 13:18:32
tags:
    - Android
    - Java
    - Gradle
---

我又来给大模型贡献素材了！

### 问题

在更新了Android Studio Narwhal Feature Drop | 2025.1.2 Patch 1版本之后，遇到了一个问题，很烦人！

> AS每次更新都能搞出点新毛病，真的服了。

使用离线依赖aar包引入某个库之后，在编辑器中按住Command查看此库的代码，会立即弹出一个Build错误提示，内容如下：

```shell
Initialization script '/private/var/folders/mw/7l7t1k7x6277h8_0jbg89z9r0000gn/T/ijArtifactDownloader1.gradle' line: 41

Execution failed for task ':app:ijDownloadArtifact8fa437a9-6cc'.
> Could not resolve all files for configuration ':app:downloadArtifact_96341a82-71d7-42e4-9f26-18fd8f441dc9'.
   > Could not find ./app/libs/BRV-1.6.1.aar:sources:.
     Required by:
         project :app

Possible solution:
 - Declare repository providing the artifact, see the documentation at https://docs.gradle.org/current/userguide/declaring_repositories.html

Ask Gemini
```

意思就是找不到这个库的源码，因为是直接放在libs下面的aar包，大概率是没有源代码的，只能看到class反编译后的Java代码，和在线依赖会自动下载source.jar（如果有）是不一样的。

其实并不是所有aar都必须带着源码发布啊，IDE自动反编译的代码已经够看逻辑了，为什么非要傻不拉几的去查找source呢？而且每点一次查看，就会弹一次错误，虽然不影响项目编译，但影响心情。

### 解决

先尝试Google，没什么结果，然后问了一下GPT，让我找到Settings中的这个路径：`Build, Execution, Deployment` → `Build Tools` → `Gradle`，把“Download external annotations”关掉。我发现这个默认本来就是关的，所以应该不是这里的问题。

再尝试Invalidate Caches清IDE缓存，也没什么用。

最终只能靠自己探索了，还是打开Settings，找到`Advanced Settings`，其中有一个“Automatically download sources for a file upon open”，把它关掉，即可解决。就是它，默认会在你打开三方库aar代码文件时，自动尝试查找和下载源码，如果失败就会弹出错误。

> 再次吐槽一下，真不知道开发IDE的人怎么想的，默认搞这个设置，不考虑开发者实际的体验。
