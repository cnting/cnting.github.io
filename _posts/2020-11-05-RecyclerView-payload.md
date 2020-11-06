---
layout:     post
title:      "为什么RecyclerView局部刷新不会闪？"
subtitle:   " 局部刷新源码解析过程来啦"
date:       2020-11-05 12:00:00
author:     "cnting"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - Android
    - RecyclerView
---



##### 0.前置知识点

首先，我们知道局部刷新是通过下面这个方法

```java
adapter.notifyItemChanged(int position,Object payload)
```

为什么用这种方式刷新不会闪一下呢？是因为这种情况下没有执行动画。后面内容从源码里找答案

此外，对 [RecyclerView的缓存机制](https://tylerliu.top/2020/05/15/Android-RecyclerView%E7%9A%84%E7%BC%93%E5%AD%98%E6%9C%BA%E5%88%B6/) 和 [RecyclerView布局过程](https://juejin.im/post/6844903937892417549) 需要有一定的了解

##### 1. RecyclerViewDataObserver#onItemRanged

```java
@Override
public void onItemRangeChanged(int positionStart, int itemCount, Object payload) {
  assertNotInLayoutOrScroll(null);
  //将更新动画添加到 mPendingUpdates 中
  if (mAdapterHelper.onItemRangeChanged(positionStart, itemCount, payload)) {
    //见流程2
    triggerUpdateProcessor();
  }
}
```

##### 2. RecyclerViewDataObserver#triggerUpdateProcessor

```java
void triggerUpdateProcessor() {
  if (POST_UPDATES_ON_ANIMATION && mHasFixedSize && mIsAttached) {
    //最终调用AdapterHelper#preProcess，然后走三个step
    RecyclerView.this.postOnAnimation(mUpdateChildViewsRunnable);
  } else {
    mAdapterUpdateDuringMeasure = true;
    //最终走三个step，在step1中也会调用AdapterHelper#preProcess
    requestLayout();
  }
}
```

##### 3. AdpaterHelper#preProcess

```java
//调用链：
//->AdapterHelper#preProcess 
//->AdapterHelper#applyUpdate
//->AdapterHelper#postponeAndUpdateViewHolders
//->RecyclerView#viewRangeUpdate

void viewRangeUpdate(int positionStart, int itemCount, Object payload) {
    final int childCount = mChildHelper.getUnfilteredChildCount();
    final int positionEnd = positionStart + itemCount;

    for (int i = 0; i < childCount; i++) {
        final View child = mChildHelper.getUnfilteredChildAt(i);
        final ViewHolder holder = getChildViewHolderInt(child);
        if (holder == null || holder.shouldIgnore()) {
            continue;
        }
        if (holder.mPosition >= positionStart && holder.mPosition < positionEnd) {
            //重点！添加FLAG_UPDATE，并且传入payload
            holder.addFlags(ViewHolder.FLAG_UPDATE);
            holder.addChangePayload(payload);
            ((LayoutParams) child.getLayoutParams()).mInsetsDirty = true;
        }
    }
    mRecycler.viewRangeUpdate(positionStart, itemCount);
}
```

##### 5. 重走三个step

在流程2最后会重走三个step，先来说下它们的区别

| 方法名                | 作用                                                         |
| --------------------- | ------------------------------------------------------------ |
| `dispatchLayoutStep1` | 处理Adapter更新，决定执行的动画效果，保存当前view的信息。如果有需要，运行预布局并保存它的信息 |
| `dispatchLayoutStep2` | 根据最终状态执行布局                                         |
| `dispatchLayoutStep3` | 执行动画                                                     |

##### 6. RecyclerView#dispatchLayoutStep1

```java
private void dispatchLayoutStep1() {
  //判断是否执行mAdapterHelper#preProcess()
  processAdapterUpdatesAndSetAnimationFlags();
  //此时mInPreLayout为true
  mState.mInPreLayout = mState.mRunPredictiveAnimations;
  ...

  if (mState.mRunSimpleAnimations) {
    // Step 0: Find out where all non-removed items are, pre-layout
    int count = mChildHelper.getChildCount();
    for (int i = 0; i < count; ++i) {
      final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
      if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
        continue;
      }
      final ItemHolderInfo animationInfo = mItemAnimator
        .recordPreLayoutInformation(mState, holder,
                                    ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                                    holder.getUnmodifiedPayloads());
      mViewInfoStore.addToPreLayout(holder, animationInfo);
      if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
          && !holder.shouldIgnore() && !holder.isInvalid()) {
        long key = getChangedHolderKey(holder);
        //将老的ViewHolder保存起来，后面有用到
        mViewInfoStore.addToOldChangeHolders(key, holder);
      }
    }
  }
  if (mState.mRunPredictiveAnimations) {
    ...
    //如果有需要，执行预布局，此时mInPreLayout=true，见流程7
    mLayout.onLayoutChildren(mRecycler, mState);
    ...
  }  
  mState.mLayoutStep = State.STEP_LAYOUT;
}
```

##### 7. LinearLayoutManager#onLayoutChildren

```java
@Override
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
  ...
  //回收当前页面上的ViewHolder，见流程8
  detachAndScrapAttachedViews(recycler);
  ...
  if (mAnchorInfo.mLayoutFromEnd) {
    //进行布局，见流程11
    fill(recycler, mLayoutState, state, false);
    ...
  }
}
```

###### 引申问题1：为什么要将ViewHolder回收到Recycler了，然后再从Recycler取？

因为`LayoutManager`负责布局，`Recycler`负责管理`ViewHolder`，`LayoutManager`不应该去了解当前需要的`ViewHolder`从哪里来

###### 引申问题2：为什么要分mChangedScrap 和 mAttachedScrap

`mChangedScrap`和`mAttachedScrap`都是在布局期间使用的。

在回收时，`mChangedScrap`保存的是会变化的ViewHolder，`mAttachedScrap`保存的是不变的ViewHolder。

在布局时，不变的ViewHolder直接从`mAttachedScrap`里取，不用重新绑定。有变化的ViewHolder，在预布局`inPreLayout`时从`mChangedScrap`里取出老的ViewHolder，然后在正常布局时从其他缓存区取出新的ViewHolder，然后比较这两个ViewHolder，进行动画



##### 8. LayoutManager#detachAndScrapAttachedViews

```java
public void detachAndScrapAttachedViews(Recycler recycler) {
  final int childCount = getChildCount();
  for (int i = childCount - 1; i >= 0; i--) {
    final View v = getChildAt(i);
    scrapOrRecycleView(recycler, i, v);
  }
}

private void scrapOrRecycleView(Recycler recycler, int index, View view) {
  final ViewHolder viewHolder = getChildViewHolderInt(view);
  if (viewHolder.shouldIgnore()) {
    if (DEBUG) {
      Log.d(TAG, "ignoring view " + viewHolder);
    }
    return;
  }
  if (viewHolder.isInvalid() && !viewHolder.isRemoved()
      && !mRecyclerView.mAdapter.hasStableIds()) {
    removeViewAt(index);
    //不能复用时，将ViewHolder移到mCachedViews 或 RecyclerViewPool
    recycler.recycleViewHolderInternal(viewHolder);
  } else {
    detachViewAt(index);
    //将ViewHolder缓存到 mAttachedScrap 或 mChangedScrap，见流程9
    recycler.scrapView(view);
    mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
  }
}
```

##### 9. Recycler#scrapView

```java
void scrapView(View view) {
  final ViewHolder holder = getChildViewHolderInt(view);
  //canReuseUpdatedViewHolder()这里是关键，见流程10
  if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
      || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
    ...
    mAttachedScrap.add(holder);
  } else {
    ...
    mChangedScrap.add(holder);
  }
}
```

##### 10. RecyclerView#canReuseUpdatedViewHolder

```java
boolean canReuseUpdatedViewHolder(ViewHolder viewHolder) {
  return mItemAnimator == null || mItemAnimator.canReuseUpdatedViewHolder(viewHolder,                                                                     viewHolder.getUnmodifiedPayloads());
}
```

这里调用了`ItemAnimator`的方法，见`DefaultItemAnimator`的实现

```java
@Override
public boolean canReuseUpdatedViewHolder(@NonNull ViewHolder viewHolder,
                                         @NonNull List<Object> payloads) {
  return !payloads.isEmpty() || super.canReuseUpdatedViewHolder(viewHolder, payloads);
}
```

可以看到，**当`payload`不为空时，是缓存到`mAttachedScrap`中的**，这是重点

##### 11. LinearLayoutManager#fill

在填充view时，会从Recycler中获取ViewHolder，最后会走到`tryGetViewHolderForPositionByDeadline()`

```java
@Nullable
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                                                 boolean dryRun, long deadlineNs) {
  //1. 预布局，根据position从mChangedScrap取
  if (mState.isPreLayout()) {
    holder = getChangedScrapViewForPosition(position);
    fromScrapOrHiddenOrCache = holder != null;
  }
  //2. 根据position 从mAttachedScrap、mHiddenViews、mCachedViews
  if (holder == null) {
    holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    ...
  }
  if (holder == null) {
    ...
    final int type = mAdapter.getItemViewType(offsetPosition);
    // 3. 根据id 和 ViewType 从 mAttachedScrap、mCachedViews取
    if (mAdapter.hasStableIds()) {
      holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
      ...
    }
    if (holder == null && mViewCacheExtension != null) {
      // 4. 从mViewCacheExtension取
      final View view = mViewCacheExtension
        .getViewForPositionAndType(this, position, type);
      ...
    }
    if (holder == null) { // fallback to pool
      // 5. 根据ViewType 从RecyclerViewPool取
      holder = getRecycledViewPool().getRecycledView(type);
      ...
    }
    if (holder == null) {
      ...
      //6. 创建ViewHolder
      holder = mAdapter.createViewHolder(RecyclerView.this, type);
    }
  }

  boolean bound = false;
  if (mState.isPreLayout() && holder.isBound()) {
    // do not update unless we absolutely have to.
    holder.mPreLayoutPosition = position;
  } else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
    //7.绑定ViewHolder
    final int offsetPosition = mAdapterHelper.findPositionOffset(position);
    bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
  }
  ...
  return holder;
}
```

从流程10里得出`ViewHolder`是存在`mAttachedScrap`中的，所以这边从`getScrapOrHiddenOrCachedHolderForPosition()`中**取出的ViewHolder和之前存放的ViewHolder是同一个**。这是重点

##### 12. RecyclerView#dispatchLayoutStep3

```java
private void dispatchLayoutStep3() {
  if (mState.mRunSimpleAnimations) {
    for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
      //获取新的ViewHolder
      ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
      if (holder.shouldIgnore()) {
        continue;
      }
      long key = getChangedHolderKey(holder);
      final ItemHolderInfo animationInfo = mItemAnimator
        .recordPostLayoutInformation(mState, holder);
      //获取老的ViewHolder
      ViewHolder oldChangeViewHolder = mViewInfoStore.getFromOldChangeHolders(key);
      if (oldChangeViewHolder != null && !oldChangeViewHolder.shouldIgnore()) {
        ...
            //关键点
            animateChange(oldChangeViewHolder, holder, preInfo, postInfo,
                          oldDisappearing, newDisappearing);
        }
      } 
  ...
}
```

因为能拿到老的ViewHolder，所以会走`animateChange()`

```java
private void animateChange(@NonNull ViewHolder oldHolder, @NonNull ViewHolder newHolder,
                           @NonNull ItemHolderInfo preInfo, @NonNull ItemHolderInfo postInfo,
                           boolean oldHolderDisappearing, boolean newHolderDisappearing) {
  oldHolder.setIsRecyclable(false);
  if (oldHolderDisappearing) {
    addAnimatingView(oldHolder);
  }
  //关键点，从流程11知道我们拿到的ViewHolder是同一个，所以不会调用 addAnimatingView()
  if (oldHolder != newHolder) {
    if (newHolderDisappearing) {
      addAnimatingView(newHolder);
    }
    oldHolder.mShadowedHolder = newHolder;
    // old holder should disappear after animation ends
    addAnimatingView(oldHolder);
    mRecycler.unscrapView(oldHolder);
    newHolder.setIsRecyclable(false);
    newHolder.mShadowingHolder = oldHolder;
  }
  //关键点，如果这个方法返回false,则不执行动画，见流程13
  if (mItemAnimator.animateChange(oldHolder, newHolder, preInfo, postInfo)) {
    postAnimationRunner();
  }
}
```

##### 13. DefaultItemAnimator#animateChange

```java
@Override
public boolean animateChange(ViewHolder oldHolder, ViewHolder newHolder,
                             int fromX, int fromY, int toX, int toY) {
  //两个ViewHolder相同
  if (oldHolder == newHolder) {
    return animateMove(oldHolder, fromX, fromY, toX, toY);
  }
  ...
}

@Override
public boolean animateMove(final ViewHolder holder, int fromX, int fromY,
                           int toX, int toY) {
  final View view = holder.itemView;
  fromX += holder.itemView.getTranslationX();
  fromY += holder.itemView.getTranslationY();
  resetAnimation(holder);
  int deltaX = toX - fromX;
  int deltaY = toY - fromY;
  //因为是局部刷新，位置没变，所以deltaX 和 deltaY 都为0，返回false
  if (deltaX == 0 && deltaY == 0) {
    dispatchMoveFinished(holder);
    return false;
  }
  ...
}
```

因为`animateChange()`返回false，导致后面动画流程不再执行，所以不会有闪的效果

##### 14. 再次梳理

1. 调用`Adapter.notifyItemChanged(position,payload)`时，会将`payload`保存
2. 在`LayoutManager`布局时，会先将页面上的`ViewHolder`都先回收，因为`payload`不为空，所以会回收到`mAttachedScrap`中
3. 然后在真正布局时，会从`mAttachedScrap`中取`ViewHolder`，和之前回收的`ViewHolder`是同一个对象
4. 在执行动画时，如果`oldHolder`和`newHolder`是同一个，并且位置没变，则不执行动画

### 参考

* [看完这篇，面试RecyclerView的时候再也不怕了](https://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650246415&idx=1&sn=14d2b70485c93f8dde664f385e8f89af&chksm=88637c60bf14f5767c65d5797e3eb5a5b73376d4746e50c71e978dff19ec5db87602b33061fb&scene=21)
* [Android-RecyclerView的缓存机制](https://tylerliu.top/2020/05/15/Android-RecyclerView%E7%9A%84%E7%BC%93%E5%AD%98%E6%9C%BA%E5%88%B6/)

* [RecyclerView 源码分析(五) - Adapter的源码分析](https://juejin.im/post/6844903937900822542#heading-5)

