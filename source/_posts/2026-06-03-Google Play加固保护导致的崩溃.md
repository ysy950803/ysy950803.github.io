---
title: Google Play加固保护导致的崩溃
date: 2026-06-03 22:21:49
tags:
  - Android
  - 问题不大
---

### 背景

事情是这样的，最近在Google Play上发布应用新版本之后，大量用户反馈启动就崩溃，吓得我赶紧停止发布。

### 分析

首先看了下改动的代码，根本和启动逻辑无关，就是功能模块一些很小的修改，并且本地编译APK安装并不会崩溃，这就很奇怪了！

因为上一个版本并不会崩溃，所以尝试回退代码，再发布到内测渠道（内测用户只有我自己）试试看，没想到还是会崩溃！那就说明和最新版的代码改动并没有关系，直接看堆栈，发现是自定义so库（`libxxx.so`）中的崩溃：

```shell
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  *** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  Build fingerprint: 'Redmi/miro/miro:16/BP2A.250605.031.A3/OS3.0.303.0.WOMCNXM:user/release-keys'
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  Revision: '0'
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  ABI: 'arm64'
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  Timestamp: 2026-06-02 16:33:29.920292745+0800
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  Process uptime: 67s
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  Cmdline: com.xxx.xxx
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  pid: 24881, tid: 24881, name: xxx.xxx  >>> com.xxx.xxx <<<
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  uid: 10608
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  tagged_addr_ctrl: 0000000000000001 (PR_TAGGED_ADDR_ENABLE)
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  pac_enabled_keys: 000000000000000f (PR_PAC_APIAKEY, PR_PAC_APIBKEY, PR_PAC_APDAKEY, PR_PAC_APDBKEY)
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A  signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x00000000aefbdede
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A      x0  00000000aefbdede  x1  000000005c000000  x2  b4000071eb03e7b0  x3  0000000000000005
2026-06-02 16:33:30.019 25823-25823 DEBUG                   crash_dump64                         A      x4  0000000002914ea8  x5  000000003b4e2f62  x6  000000003b4e2f62  x7  0000007090d52d9c
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A      x8  00000000eefdbdea  x9  7b7e8f863744d50d  x10 0000000000000000  x11 0000000000000008
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A      x12 0000000000000080  x13 00000073efa90000  x14 0000000000000080  x15 00000073efa90200
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A      x16 0000000000000001  x17 000000710fb4198c  x18 00000073f0790000  x19 b40000729b042ab0
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A      x20 0000007fcde918d0  x21 00000073efa9001d  x22 00000073efa90025  x23 0000000000000000
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A      x24 00000073efdeb780  x25 0000000000000000  x26 0000000010300011  x27 000000000000002c
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A      x28 0000007fcde91940  x29 0000007fcde91a80
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A      lr  000000710a736524  sp  0000007fcde918d0  pc  000000710a736608  pst 0000000060001000
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A  1 total frames
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A  backtrace:
2026-06-02 16:33:30.020 25823-25823 DEBUG                   crash_dump64                         A        #00 pc 0000000000025608  /data/app/~~nZXQOJY2UsliNvUCLuNzTg==/com.xxx.xxx-9W8qkHb9TVlLIY-xrqiJzA==/split_config.arm64_v8a.apk!libxxx.so (offset 0x4000) (BuildId: 212ce503ae1e13b86cdeafa62340b31f3efb6a12)
2026-06-02 16:33:30.032 25823-25823 MIUINDBG                crash_dump64                         E  miui_native_debug_process_O
2026-06-02 16:33:30.033 25823-25823 MIUINDBG                crash_dump64                         E  crash_dump(miuindbg) client read ack success
2026-06-02 16:33:30.033 25823-25823 MIUINDBG                crash_dump64                         E  core type: 0 
2026-06-02 16:33:30.033 25823-25823 MIUINDBG                crash_dump64                         E  unable to connect to mqsas native socket
```

分析堆栈后发现是触发了我之前自己做的一个Application类防篡改机制，即在应用启动时会检查当前ApplicationContext对应的实例是否被替换，若类名不正确，就会在native代码中抛出异常强制让应用崩溃。

那么问题就变成了：为什么应用的Application入口类被篡改了呢？Google Play开发者后台在优化分发aab包的时候做了什么？

直接反编译看看吧，发现果然被改成了这样：

```java
package com.pairip.application;

import android.content.Context;
import com.pairip.licensecheck.LicenseClient;

public class Application extends MyApp {
    @Override
    public void attachBaseContext(Context context) {
        LicenseClient.checkLicense(context);
        super.attachBaseContext(context);
    }
}
```

`MyApp`是我自己的Application入口类，传到Play上后，被修改为了`com.pairip.application.Application`来继承应用本身的入口类，所以触发了应用本身的防篡改机制。

查了一下资料，`com.pairip.application.Application` 是谷歌（Google）第三方License（数字授权）校验及加固保护机制的核心组件之一。它主要用于验证Android应用是否为正版授权，防止软件被篡改或盗版。

好家伙，谷歌悄悄咪咪搞事情，不通知开发者，我确实早就开启了Play的应用保护机制，但以前不会直接这么篡改应用包本身的内容。对Application入口进行偷梁换柱也是破解应用的手法之一，谷歌算是自己当流氓来治流氓了。

### 解决

既然问题搞清楚了，解决也简单，在自定义的防篡改机制判断中，把谷歌这个类名加入白名单判断即可。
