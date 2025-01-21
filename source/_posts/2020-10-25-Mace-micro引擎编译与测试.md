---
layout:     post
title:      Mace-micro引擎编译与测试
subtitle:   移动端AI计算框架。
date:       2020-10-25
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - AI
    - 鼓捣折腾
    - Android
---

![](https://github.com/XiaoMi/mace/raw/master/docs/mace-logo.webp)

## 官方简介

**Mobile AI Compute Engine (MACE)** 是一个专为移动端异构计算平台(支持Android, iOS, Linux, Windows)优化的神经网络计算框架。
主要从以下的角度做了专门的优化：

* 性能
  * 代码经过NEON指令，OpenCL以及Hexagon HVX专门优化，并且采用[Winograd算法](https://arxiv.org/abs/1509.09308)来进行卷积操作的加速。
    此外，还对启动速度进行了专门的优化。
* 功耗
  * 支持芯片的功耗管理，例如ARM的big.LITTLE调度，以及高通Adreno GPU功耗选项。
* 系统响应
  * 支持自动拆解长时间的OpenCL计算任务，来保证UI渲染任务能够做到较好的抢占调度，从而保证系统UI的相应和用户体验。
* 内存占用
  * 通过运用内存依赖分析技术，以及内存复用，减少内存的占用。另外，保持尽量少的外部依赖，保证代码尺寸精简。
* 模型加密与保护
  * 模型保护是重要设计目标之一。支持将模型转换成C++代码，以及关键常量字符混淆，增加逆向的难度。
* 硬件支持范围
  * 支持高通，联发科，以及松果等系列芯片的CPU，GPU与DSP(目前仅支持Hexagon)计算加速。CPU模式支持Android, iOS, Linux等系统。
* 模型格式支持
  * 支持[TensorFlow](https://github.com/tensorflow/tensorflow)，[Caffe](https://github.com/BVLC/caffe)和[ONNX](https://github.com/onnx/onnx)等模型格式。

## Micro Controllers

上面的简介摘自[官方GitHub介绍](https://github.com/XiaoMi/mace/blob/master/README_zh.md)，可以让大家基本了解什么是mace引擎，它是一个移动端的AI计算框架。随着谷歌Tensorflow Lite的推广，越来越多的移动应用都将部分AI计算能力迁移至端侧，这些机器学习的算法都是运行在应用层的。但大多数系统层面的计算，例如省电优化（根据应用使用习惯）、睡眠监测、运动行为识别、语音唤醒手机等，为了保证低功耗，必须放在底层，这样计算过程中就无需唤醒上层的一系列进程，只等有结果时才唤醒并上报数据。

有多底层呢？通常来说就是BP侧，在HAL层以下，逻辑基本运行在类似DSP这样的处理器上，部分工作需要开发人员和硬件驱动打交道。mace框架支持将TF模型文件转换成C++代码并打包成micro组件，方便我们集成。

## 编译

官方文档（[https://mace.readthedocs.io/en/latest/micro-controllers/basic_usage.html](https://mace.readthedocs.io/en/latest/micro-controllers/basic_usage.html)）写得比较详尽，但一些细节和注意点还是只有实践过程中才会发现。我的系统是Ubuntu 20.04，部分依赖安装起来稍微麻烦一点（mace官方要求的依赖很多版本很旧）。

先配置基础环境，要装的东西挺多的，按官方给的版本号安装即可（[https://mace.readthedocs.io/en/latest/installation/env_requirement.html](https://mace.readthedocs.io/en/latest/installation/env_requirement.html)）。下面只提一些注意点。

1、安装Python 2.7：Ubuntu 18.04就不自带Python 2.x了，默认是3.x，所以一定要自己手动安装Python 2.7，注意配置python和pip命令默认对应的版本为2.x。虽然mace官方文档写的3.6也可以编，但是我试过，各种失败。

2、安装Bazel 0.13.x（实测0.13.0和0.13.1都行）：由于Ubuntu直接用apt安装bazel的话版本会非常高，最后会导致编译失败。Ubuntu的话直接到bazel官方GitHub下载对应版本的deb文件来安装即可：[https://github.com/bazelbuild/bazel/releases?after=0.16.1](https://github.com/bazelbuild/bazel/releases?after=0.16.1)

```bash
sudo dpkg -i bazel_0.13.1-linux-x86_64.deb
```

3、由于我只在手机BP侧运行micro代码，且基于Tensorflow的模型文件转换，所以官方文档中提到的Android NDK、ADB和Docker这里都不需要安装，除此之外都最好安装完。

配置好各种环境后，就可以根据basic_usage中的步骤拉取mace代码，并执行转换命令：

```bash
# 直接执行，不需要改
git clone https://github.com/XiaoMi/mace.git
cd mace/
git fetch --all --tags --prune
tag_name=`git describe --abbrev=0 --tags`
git checkout tags/${tag_name}
# 转换脚本（保证是Python 2.x版本）
python tools/python/convert.py --config=xxx.yml --enable_micro
```

这里的xxx.yml是模型转换的配置文件，可以参考官方示例：[https://github.com/XiaoMi/mace-models](https://github.com/XiaoMi/mace-models)

转换成功后，在mace/build/xxx/model目录下面会有转换成功的产物，解压xxx.tar.gz压缩包即可得到micro代码。

## 测试

如果每次都将micro代码集成到底层打包再进行模型测试，那太麻烦了，效率低下。因此，我们可以将其直接运行在PC上来进行输入数据mock，这个是官方文档没有提到的，需要我们自己修改CMake配置文件并添加测试代码。

我用CLion直接打开micro文件夹，导入后会自动加载CMakeLists文件，然而肯定是编不过的，需要对各目录下的CMakeLists稍作修改，可以直接根据错误提示来。由于不同模型转换得到的C++代码可能会有一些差异，但大致应该是差不多的，比如我这个模型转换后就只需要修改ops/CMakeLists.txt，注释掉其中不存在的源文件（因为你的模型可能不需要这些代码，所以刚才的convert脚本给你删掉了，但CMakeLists里又没做调整，应该是脚本没考虑周全吧）。示例：

```cmake
set(MICRO_OPS_SRCS
#  shape.cc
#  reduce.cc
  reshape.cc
  matmul.cc
#  nhwc/depthwise_conv_2d_ref.cc
#  nhwc/conv_2d_c4_s4.cc
#  nhwc/depthwise_conv_2d_kb3_s4.cc
#  nhwc/pooling_ref.cc
#  nhwc/conv_2d_c3_s4.cc
#  nhwc/conv_2d_ref.cc
  nhwc/depthwise_conv_2d_kb4_s4.cc
  nhwc/depthwise_conv_2d_kb1_s4.cc
  nhwc/base/depthwise_conv_2d_base.cc
  nhwc/base/conv_2d_base.cc
  nhwc/base/filter_op_base.cc
  nhwc/base/pooling_base.cc
#  nhwc/depthwise_conv_2d_kb2_s4.cc
#  nhwc/conv_2d_c2_s4.cc
#  nhwc/batch_norm.cc
  nhwc/pooling_s4.cc
  bias_add.cc
  softmax.cc
#  eltwise.cc
  utils/gemm.cc
  utils/crumb_utils.cc
  utils/gemv.cc
  utils/activation.cc
#  expand_dims.cc
#  squeeze.cc
  activation.cc
)

add_library(micro_ops
  ${MICRO_OPS_SRCS}
)
target_link_libraries(micro_ops
  micro_base
  micro_framework
)

add_library(micro_ops_for_test
  ${MICRO_OPS_SRCS}
)
target_link_libraries(micro_ops_for_test
  micro_base
  micro_framework_for_optest
)
target_compile_options(micro_ops_for_test PUBLIC -fPIC)
```

编译错误提示找不到的，直接注释掉即可。接下来就可以自己编写测试代码来验证模型输出了，具体可以参考官方文档basic_usage最后一节。比如我这里在micro目录下添加了一个主入口测试文件main_test.cc和一个依赖文件my_test_utils.cc（包括.h头文件），只需要修改一下同目录下的CMakeLists（这个文件也是micro的顶层编译配置）即可：

```cmake
# 追加配置即可
# main_test_srcs、micro_test_run_lib、micro_test_run都是自定义的名字
file(GLOB main_test_srcs my_test_utils.cc)
add_library(micro_test_run_lib
  ${main_test_srcs}
)
add_executable(micro_test_run main_test.cc)
# 这里必须链接micro_engine_c，这样才能调用mace的API
target_link_libraries(micro_test_run
  micro_test_run_lib
  micro_engine_c
)
```
