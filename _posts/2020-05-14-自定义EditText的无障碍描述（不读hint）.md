---
layout:     post
title:      自定义EditText的无障碍描述（不读hint）
subtitle:   探索发现。
date:       2020-05-14
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Java
    - Android
---

## 问题

我们一般给一个控件设置描述时，会这样：

```java
xxxView.setContentDescription("xxx");
```

但是，当你给EditText设置这个时，会发现毫无卵用。为什么呢？

搜了下EditText和其直接父类TextView，没有重写setContentDescription方法，那应该不是setXXX时发生改变，而是getXXX的问题。

果然，在TextView中发现：

```java
/**
 * Returns the text that should be exposed to accessibility services.
 * <p>
 * This approximates what is displayed visually. If the user has specified
 * that accessibility services should speak passwords, this method will
 * bypass any password transformation method and return unobscured text.
 *
 * @return the text that should be exposed to accessibility services, may
 *         be {@code null} if no text is set
 */
@Nullable
@UnsupportedAppUsage
private CharSequence getTextForAccessibility() {
    // If the text is empty, we must be showing the hint text.
    if (TextUtils.isEmpty(mText)) {
        return mHint;
    }
 
    // Otherwise, return whatever text is being displayed.
    return TextUtils.trimToParcelableSize(mTransformed);
}
```

所以EditText在获取到无障碍焦点时，只会朗读hint文本，而不是contentDescription。其实这个设计是没有问题的，可编辑控件，在没有输入内容时，就应该朗读hint。

但是，某些自定义控件是长这样的：

![MIUI EditText](https://img-blog.csdnimg.cn/2020051423404290.png)

label是自定义View画上去的，没做特殊处理的情况下Talkback识别不到，最好的体验是把左边的label也跟着读出来（比如读成：“列车车次，例：G1”），这可咋办？

## 解决

很显然，不能直接去改hint，否则UI显示不对。

### 尝试一：获取焦点时我自己读一串文本行不行

我们知道，可以通过：

```java
xxxView.announceForAccessibility("xxx");
```

来进行无障碍朗读，但是并没有一个类似setOnFocusChangeListener的方法来专门监听无障碍焦点，所以这个不好搞。

### 尝试二：Read the fucking code

其实无障碍开发中还有一些关键方法，且Talkback这些无障碍辅助工具最终其实也会触发这些方法的：

```java
xxxView.requestAccessibilityFocus(); // 获取无障碍焦点，自动朗读已设置的描述
xxxView.sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUSED); // 效果和前者差不多，且前者最后也要调用此方法
```

深入后可以跟踪到：

```java
public void sendAccessibilityEventUncheckedInternal(AccessibilityEvent event) {
    ...
    onInitializeAccessibilityEvent(event);
    // Only a subset of accessibility events populates text content.
    if ((event.getEventType() & POPULATING_ACCESSIBILITY_EVENT_TYPES) != 0) {
        dispatchPopulateAccessibilityEvent(event);
    }
    // In the beginning we called #isShown(), so we know that getParent() is not null.
    ViewParent parent = getParent();
    if (parent != null) {
        getParent().requestSendAccessibilityEvent(this, event);
    }
}
```

这个onInitializeAccessibilityEvent的源码注释写得很明白，就是控件获取到无障碍事件时会触发，但通过event参数我们貌似做不了什么。

在Textview中我们发现，与onInitializeAccessibilityEventInternal相邻有一个内部方法 **onInitializeAccessibilityNodeInfoInternal**：

```java
public void onInitializeAccessibilityNodeInfoInternal(AccessibilityNodeInfo info) {
    super.onInitializeAccessibilityNodeInfoInternal(info);
 
    final boolean isPassword = hasPasswordTransformationMethod();
    info.setPassword(isPassword);
    info.setText(getTextForAccessibility());
    info.setHintText(mHint);
    info.setShowingHintText(isShowingHint());
    ...
}
```

这个 **info.setText(getTextForAccessibility());** 就是关键了，它其实才是最终朗读出来的那个文本。

公共方法onInitializeAccessibilityNodeInfo的注释也说明了：Initializes an {@link AccessibilityNodeInfo} with information about this view. 此方法初始化一些View的无障碍基本信息。

**最终解决：**

在自定义的EditText类中重写方法，覆盖文本，这样在朗读时就是自己想要的了：

```java
@Override
public void onInitializeAccessibilityNodeInfo(AccessibilityNodeInfo info) {
    super.onInitializeAccessibilityNodeInfo(info);
    // 对于EditText，系统无障碍朗读只读hint，需通过节点info覆盖自定义内容
    info.setText("xxx" + getHint());
}
```

Tips：其实这里为了API统一，我是直接 **info.setText(getContentDescription());** 方便很多。
