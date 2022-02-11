---
layout:     post
title:      Rust为Android应用编译so库
subtitle:   新东西来啦！
date:       2022-02-11
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
    - Rust
---

### 前言

Rust是个好东西，Google也开始把它用于AOSP了。我们开发应用同样也可以使用Rust来编写原本为C++的Native代码。网上搜罗一圈，入门的文档不多不少，这里稍微归纳整理一下吧，毕竟Hello World是人类的一大步。

### 安装Rust

Rust的文档真的非常棒，目前的翻译版本也几乎满足所有学习需求。安装很简单，参考官网（[Rust-lang.org](https://www.rust-lang.org/zh-CN/learn/get-started)）即可，一行命令：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### 配置NDK

1、先确保你在Android Studio的SDK Manager中下载安装好了NDK相关的工具包，基操就不赘述了。

2、默认目录一般都在 `/Users/你的用户名/Library/Android/sdk/ndk-bundle` 这个位置，用户目录可以用 `${HOME}` 代替。当然，如果你的SDK在其他位置，按你的来即可。

在任意处创建一个名为NDK的目录（名字随意，也可不叫NDK），然后运行NDK工具包中的py脚本以编译NDK开发环境：

```bash
cd ~
mkdir NDK
# 不同架构参数不同，按需配置即可，比如我就只需要arm64
# api参数最好按你的应用targetSDK的版本号来，比如我这里是30
# Python版本我这里是3.8，如果你是Python 2.x的话，不确定能否运行成功
python ${HOME}/Library/Android/sdk/ndk-bundle/build/tools/make_standalone_toolchain.py --api 30 --arch arm64 --install-dir NDK/arm64
python ${HOME}/Library/Android/sdk/ndk-bundle/build/tools/make_standalone_toolchain.py --api 30 --arch arm --install-dir NDK/arm
python ${HOME}/Library/Android/sdk/ndk-bundle/build/tools/make_standalone_toolchain.py --api 30 --arch x86 --install-dir NDK/x86
```

3、编辑Rust环境的配置文件即 `~/.cargo/config` ，若无，新建即可，并添加内容：

```bash
# 同样是按需配置，如果你不需要编译其他架构，就不添加
# 相关路径最好写绝对路径，此处若用${HOME}不生效
[target.aarch64-linux-android]
ar = "/Users/你的用户名/NDK/arm64/bin/aarch64-linux-android-ar"
linker = "/Users/你的用户名/NDK/arm64/bin/aarch64-linux-android-clang"

[target.armv7-linux-androideabi]
ar = "/Users/你的用户名/NDK/arm/bin/arm-linux-androideabi-ar"
linker = "/Users/你的用户名/NDK/arm/bin/arm-linux-androideabi-clang"

[target.i686-linux-android]
ar = "/Users/你的用户名/NDK/x86/bin/i686-linux-android-ar"
linker = "/Users/你的用户名/NDK/x86/bin/i686-linux-android-clang"
```

4、添加编译工具链，和第3步中配置的对应：

```bash
rustup target add aarch64-linux-android armv7-linux-androideabi i686-linux-android
```

### 编写Demo

配置好各种环境后，就可以开始Coding了，先生成一个Rust的lib空项目，`rust-android-libs` 是我的自定义命名：

```bash
cargo new rust-android-libs --lib
```

进入目录，编辑Cargo.toml配置文件，直接修改如下：

```toml
[package]
name = "rust-android-libs"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
jni = { version = "0.19.0", default-features = false }

[lib]
crate_type = ["cdylib"]
```

这里Rust的JNI版本，可以参考官方文档：[Docs.rs](https://docs.rs/releases/search?query=jni)，目前最新为0.19.0，下面的crate_type配置也是按文档来的。

3、编写Rust代码，在src/lib.rs文件中：

```rust
#![cfg(target_os = "android")]
#![allow(non_snake_case)]

use jni::JNIEnv;
use jni::objects::{JClass, JString};
use jni::sys::jstring;

#[no_mangle]
pub extern "C" fn Java_com_xxx_xxx_Yyy_getTestStr(
    env: JNIEnv, _: JClass,
) -> jstring {
    env.new_string("Hello World!")
        .expect("Couldn't create java string!")
        .into_inner()
}

#[no_mangle]
pub extern "C" fn Java_com_xxx_xxx_Yyy_getTestStrWithInput(
    env: JNIEnv, _: JClass, input: JString,
) -> jstring {
    let input: String = env.get_string(input)
        .expect("Couldn't get java string!")
        .into();
    let output = env.new_string(format!("Hello, {}!", input))
        .expect("Couldn't create java string!");
    output.into_inner()
}
```

如上所示，我们编写了两个方法，一个是直接返回一个String，另一个是带参数返回拼接后的String。方法的命名规则和写C++的JNI代码一样，以此处为例，说明我们需要在Java/Kotlin代码中对应创建一个包名为 `com.xxx.xxx`，名为 `Yyy` 的类，其中有两个方法：

```kotlin
package com.xxx.xxx

object Yyy {
    init {
        // 因为等会儿编译的so产物为librust_android_libs.so，所以此处加载命名如下
        System.loadLibrary("rust_android_libs")
    }

    external fun getTestStr(): String
    external fun getTestStrWithInput(input: String): String
}
```

4、编译Rust项目，按需要的架构编译即可，如果不用模拟器的话，一般都不用考虑x86：

```bash
cargo build --target aarch64-linux-android --release
cargo build --target armv7-linux-androideabi --release
cargo build --target i686-linux-android --release
```

编译成功后在项目的 `/target/aarch64-linux-android/release/librust_android_libs.so` 路径下可以找到想要的so文件，把它复制到Android项目中。

一般来说都在 `.../app/src/main/jniLibs/arm64-v8a/` 这样的目录下，按你需要的架构来即可。同时记得配置build.gradle：

```groovy
...
android {
    compileSdkVersion 30
    defaultConfig {
        ...

        ndk {
            abiFilters 'arm64-v8a'
        }
    }

    // 如果你的AGP插件版本不小于7.0，可能需要添加
    packagingOptions {
        jniLibs {
            useLegacyPackaging = true
        }
    }
}
...
```

### 后话

整个流程下来，还是有点小繁琐的，同时也整理几个问题吧：

1、同样的代码逻辑，Rust编译出来的so库比C++编译的要大很多，以我上面的代码为例，64位架构大约在**4MB**左右，32位在3MB多一点，而C++编译的产物分别为**1MB**多和900多KB，这对包体积大小敏感的项目来说，差距还是不小的。不知道有没有优化的方法，由于我也是刚接触Rust，还不太了解。

```bash
# 我们可以用strings命令来对比两种环境编译出来的so结构
strings librust_android_libs.so
```

2、复杂的业务逻辑肯定需要调试，以方便定位问题，但以我们上面的例子来看，Rust项目和Android是分开的，如何像C++代码那样直接在Android项目中整合编译并断点调试，还需要进一步探索。

3、Google和Rust官方对于适配到Android应用项目的相关文档还不是很丰富，我们在Studio中New Project的时候，就可以看见有Native C++的模板可以选择，这给了初学者很好的示范，希望以后也有Native Rust之类的模板。
