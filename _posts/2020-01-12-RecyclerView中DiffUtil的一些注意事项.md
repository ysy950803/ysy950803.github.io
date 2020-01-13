---
layout:     post
title:      RecyclerView中DiffUtil的一些注意事项
subtitle:   见微知著。
date:       2020-01-12
author:     YSY
header-img: img/home-bg.jpg
catalog: true
tags:
    - Android
---

### 节能刷新

移动设备屏幕大小有限（不得不说我是顽固的小屏爱好者，大于5.5寸难以接受，时代已经抛弃我了哈哈），列表（List）可以说是一个出现非常高频的交互设计。大多数情况下我们的列表不仅仅是一次性加载本地数据，而要应付来自网络的各种动态内容，可能是增加、删除等操作。

在Android开发中，一个耳熟能详的方法就是 `notifyDataSetChanged` ，在适配器（Adapter）的设计模式下，每当我们的列表数据发生变更时，就需要调用此方法来更新UI。然而，这个方法并不“**节能**”，它会同时刷新列表中的所有item，包括那些并没有变化的数据，这样就带来很多计算资源的浪费。要知道，从你的一个 `setText` 或者 `setImageResource` 方法调用到最终呈现到屏幕上，软件到硬件，中间经历了非常复杂的过程。基于能省则省的移动开发原则，有没有更好的办法呢？

### DiffUtil用起来

谷歌确实也考虑到了这个问题，所以不知道在什么时候（暂时没有去查阅）推出了DiffUtil这个解决方案。在RecyclerView的依赖包下面，可以看到，除了DiffUtil，还有异步处理数据等一系列有趣的工具。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200113015406917.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lzeTk1MDgwMw==,size_16,color_FFFFFF,t_70)
DiffUtil的运用逻辑非常简单，大致如下：

- 实现对比新旧数据的方法（类似比较器），这样DiffUtil便知道当新数据来临时，该不该更新某个item。
- 更新数据时，把新旧数据丢给DiffUtil，底层会根据你实现的对比方法，利用一种差分算法自动计算出差异，最后局部更新到UI。

这样做的好处就是避免了不必要的UI更新，DiffUtil计算出差异之后，只刷新产生变动的item。具体地，我们可以在Adapter的 `onBindViewHolder` 方法打断点或者日志观察，或者调用 `registerAdapterDataObserver` 方法监听item的各种操作情况。其次，以前的 `notifyDataSetChanged` 方法由于会刷新整个列表所以没有原生的动画效果，而DiffUtil内部最终调用了各种 `notifyItemXXX` 方法。

DiffUtil的使用也很简单：

1. 先实现比较新旧数据的回调，可以是一个独立的类，也可以写成Adapter的内部类：

```java
   public class BaseXXXAdapter<T> extends RecyclerView.Adapter {
       // ...
       
   	private class DiffCallback extends DiffUtil.Callback {
           private List<T> oldData, newData;
   
           DiffCallback(List<T> oldData, List<T> newData) {
               this.oldData = oldData;
               this.newData = newData;
           }
   
           @Override
           public int getOldListSize() {
               return oldData.size();
           }
   
           @Override
           public int getNewListSize() {
               return newData.size();
           }
   
           @Override
           public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
               T oldT = oldData.get(oldItemPosition);
               T newT = newData.get(newItemPosition);
               // 实际情况最好是在此处对比新旧数据的id（比如用户uid），这里为了方便示例直接equals对象了
               // 若此处返回true，则DiffUtil不会再调用下面的areContentsTheSame方法
               // 若此处返回false，才会继续去调下面的方法校验更多内容
               return Objects.equals(oldT, newT);
           }
   
           @Override
           public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
               // TODO 比较新旧数据是否相同，这里为了方便示例直接返回true
               return true;
           }
       }
   }
```

2. 然后在Adapter内部实现一个update数据的方法：

```java
       @Override
       public void updateData(List<T> newData) {
           DiffUtil.DiffResult result = DiffUtil.calculateDiff(new DiffCallback(getItems(), newData));
           // 这里的getData即表示获取整个列表的数据，自行实现即可
           getData().clear();
           getData().addAll(newData);
           result.dispatchUpdatesTo(this);
       }
```

   注意这里的 `dispatchUpdatesTo` 可以在clear之前，也可以在addAll之后，实际效果暂未发现什么区别，之前查阅资料包括官方示例也都是最后执行dispatch，姑且认为这样算标准吧。

3. ……咦，怎么才两步，确实就这么简单。重点还是 `areItemsTheSame` 和 `areContentsTheSame` 方法，后者大部分时候只需要对比每个item上UI展示出来的数据即可，因为用户只关心眼见的内容。

### 解决使用后产生的问题

我们会发现在上面的使用示例中，`updateData` 方法内部对原数据进行了清除和添加的操作，这会导致一个问题便是：**列表数据集合中的对象已经变了，即使其某项对应的UI内容没有发生变化**。

举个例子，一个通讯录列表里面有 **[小明, 小红]** 两个人，对应内存地址为 **[a1, a2]**，现在通过上述 `updateData` 方法更新了通讯录列表，UI内容变成了 **[小王, 小红]**，对应内存地址为 **[b1, b2]**。对用户来说小红这个item看上去没有发生变化，但其实对应的数据类对象已经不同。**而且此时 `onBindViewHolder` 方法只会触发一次，将小明更新成小王，而不会触发小红那个position对应的 `onBindViewHolder`** 。

上述细节很关键，如果开发过程中绑定（bind）数据**不恰当**的话，就容易造成各种奇异问题，比如网上资料最多的DiffUtil导致item点击事件数据错位问题、数组越界崩溃问题等等。

这里的“不恰当”，绝大部分情况下，总结出来：其实指的就是在 `onBindViewHolder` 方法中持有了某个位置（position）对应数据的不可变对象。最常见的误用示例就是在 `onBindViewHolder` 中设置某些控件的点击事件并引用数据对象：

```java
	// 此处假设item的数据类为User
	@Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
        MyItemViewHolder h = (MyItemViewHolder) holder;
        User user = getData().get(position);
        h.mNameTextView.setOnClickListener(v -> {
            // 第2种写法：User user = getData().get(position);
            // 假设这里是点击item跳转到该User对应的个人主页界面
            startWebView(user.getHomePageUrl());
        });
    }
```

在不接入DiffUtil之前，上面这段代码没有任何问题，因为我们都是使用 `notifyDataSetChanged` 方法来更新UI，每次更新调用到 `onBindViewHolder` 时，点击事件都会重新设置，get出来的user对象自然也是最新的。一旦我们使用了DiffUtil，就会出问题了。

回到上面小王绿了小明的例子，在我们的 `updateData` 方法执行后，如果我们只对比了user的名字这个属性（其实也只需要对比这个属性），那么小红那一个item就不会触发对应的 `onBindViewHolder` ，即小红的点击事件回调里，**仍然持有着旧数据集的user对象（对应那个内存地址a2）**。但实际上小红应该对应 **b2** 那个内存了，这就造成 **a2** 内存无法释放，问题是不是显得有点严重了。

有同学说无所谓呀，反正点击事件依然有效。那如果我说网络数据刷新下来小红的 **homePageUrl** 变了呢？是不是还得把这个属性加入DiffUtil的对比方法中？这样最终会导致小红的 `onBindViewHolder` 方法也执行，跟 `notifyDataSetChanged` 岂不是没什么两样了？

此外，若get对象写成注释中的第2种写法，且列表第0个位置的item被删了呢？小红顶上去变成了第0个，此时由于小红的UI内容没变，只是位置变了，所以 `onBindViewHolder` 依然不会执行。以上面的示例代码来看，当再次点击小红时，就会直接出现数组越界的异常。因为position还是之前的1，而此时小红的position已经为0。

显然上述出现的这些问题不符合谷歌的设计初衷，也不符合我们使用DiffUtil的初衷。其实**解决办法**很简单，就是要对 `onBindViewHolder` 方法有一个正确的认知，其原则就是：

- `onBindViewHolder` 只做UI内容的更新，如 `setText`，`setImageXXX` 等方法。做到数据类一次性使用。
- 不要跨作用域持有与位置（position）相关的数据，比如每个item的数据对象。尤其就是避免在 `onBindViewHolder` 中设置点击事件监听。

正确的点击事件监听还是参照如下形式比较好：

```java
// 比如这是某个Base适配器类
public class BaseXXXAdapter<T> extends RecyclerView.Adapter {
    // ...
    private View.OnClickListener mOnClickListener;
    private View.OnLongClickListener mOnLongClickListener;
    private OnItemClickListener mOnItemClickListener;

    public interface OnItemClickListener {
        void onItemClick(View view, RecyclerView.ViewHolder holder, int position);

        void onItemLongClick(View view, RecyclerView.ViewHolder holder, int position);
    }

    public BaseXXXAdapter(Context context) {
        // ...
        mOnClickListener = v -> {
            RecyclerView.ViewHolder h = (RecyclerView.ViewHolder) v.getTag();
            int pos = h.getAdapterPosition();
            if (mOnItemClickListener != null) {
                mOnItemClickListener.onItemClick(v, h, pos);
            }
        };
        mOnLongClickListener = v -> {
            RecyclerView.ViewHolder h = (RecyclerView.ViewHolder) v.getTag();
            int pos = h.getAdapterPosition();
            if (mOnItemClickListener != null) {
                mOnItemClickListener.onItemLongClick(v, h, pos);
            }
            return true;
        };
    }

    public void setOnItemClickListener(OnItemClickListener clickListener) {
        this.mOnItemClickListener = clickListener;
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        // ...省略holder实例化
        holder.itemView.setTag(holder); // 把holder当tag存
        holder.itemView.setOnClickListener(mOnClickListener);
        holder.itemView.setOnLongClickListener(mOnLongClickListener);
        return holder;
    }
}

// 继承实现的实际业务Adapter
public class XXXAdapter extends BaseXXXAdapter<User> {
    public XXXAdapter(Context context) {
        setOnItemClickListener(new OnItemClickListener() {
            @Override
            public void onItemClick(View view, RecyclerView.ViewHolder holder, int position) {
                MyItemViewHolder h = (MyItemViewHolder) holder;
                // 每次点击都保证了为对应位置的数据，再也不用担心数据错位问题了
                User user = getData().get(position);
            }

            @Override
            public void onItemLongClick(View view, RecyclerView.ViewHolder holder, int position) {
                // ...
            }
        });
    }
}
```
