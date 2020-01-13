---
layout:     post
title:      避免Android定制ROM中的EditText出现pop弹窗
subtitle:   知其所以然。
date:       2019-12-07
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
---

### 问题
可能看到标题的同学一头雾水，这是什么东西，其实类似于你长按文本时出现的复制粘贴pop弹窗。在一些定制ROM中，厂商为了某些方便用户的特殊功能会增加文本输入检测和自定义弹窗，举例：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191207163617238.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)
比如在你输入一些邮箱时，会自动弹出这种pop（并不需要你长按），且这个并非系统应用才有的，而是直接影响了所有EditText。
对于一些高度定制化UI的应用来说，这个可能影响用户体验。下面给出两种解决办法（都不算特别完美，毕竟卑微的应用层开发者改不了底层实现），大家酌情参考。
### 解决
**方法一：**
要解决问题先思考（合理猜测）一下它的实现原理，很显然它修改了EditText或者TextView源码，在文本输入监听中加入了对邮箱特征的检测，我们可以尝试修改getText方法的返回值，过滤这种特征：
```java
	// 在自定义的EditText中重写getText
	@Override
	public Editable getText() {
	    String text = super.getText() != null ? super.getText().toString() : null;
	    if (text != null && text.contains("@")) {
	    	// 如果输入内容包含@符号，则删掉再返回
	        return Editable.Factory.getInstance().newEditable(text.replaceAll("@", ""));
	    }
	    return super.getText();
	}
```
当然有同学会说，上述办法也影响了getText方法返回内容的正确性，没关系，我们反正是自定义控件，再补一个方法即可，业务代码外部调用就调这个：
```java
	public Editable getRealText() {
	    return super.getText();
	}
```
**方法二：**
这个方法适合不需要自定义EditText游标的同学，非常简单，给EditText的xml代码加上 `android:textCursorDrawable="@null"` 属性即可。
有人会觉得奇怪，这个cursor的drawable跟pop弹窗有啥关系，因为原生的复制粘贴pop弹窗在显示之前要计算游标（Cursor）的位置，且会检查 `mDrawableForCursor` 是否存在，如果不存在就不走后续逻辑了，具体可以参看Editor源码：
```java
    void updateCursorPosition() {
        loadCursorDrawable();
        if (mDrawableForCursor == null) {
            return;
        }
        // ...
    }
```
那么上述“常用邮箱”之类的弹窗其实也类似。
