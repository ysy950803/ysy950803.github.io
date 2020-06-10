---
layout:     post
title:      Android神经网络（Neural Networks API）初探
subtitle:   见微知著。
date:       2020-01-14
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
---

> 谷歌早在Android 8.0就推出了神经网络API，不过现在网上资料仍旧不多，随着TensorFlow Lite的成熟，底层API更是无人问津。

### 前言

Android Neural Networks API (NNAPI) 是一个 Android C API，专为在 Android 设备上运行计算密集型运算从而实现机器学习而设计。NNAPI 旨在为更高层级的机器学习框架（如 TensorFlow Lite 和 Caffe2）提供一个基本功能层，用来建立和训练神经网络。搭载 Android 8.1（API 级别 27）或更高版本的所有 Android 设备上都提供该 API。

### 参考

- [NNAPI介绍](https://developer.android.com/ndk/guides/neuralnetworks)
- [NNAPI文档](https://developer.android.com/ndk/guides/neuralnetworks)
- [NNAPI官方Sample](https://github.com/android/ndk-samples/tree/master/nn_sample)
- [NNAPI实现-TFLite](https://github.com/tensorflow/tensorflow/tree/master/tensorflow/lite/nnapi)

挖坑待填，未完待续……
