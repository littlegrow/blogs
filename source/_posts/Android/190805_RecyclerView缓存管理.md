---
title: RecyclerView ViewHoler 缓存机制
date: 2019-8-6 19:00:00
description: "四级缓存"
categories:
- RecyclerView
- ViewHoler缓存
---

### 四级缓存

#### 第一级

```
final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
ArrayList<ViewHolder> mChangedScrap = null;
```

mAttachedScrap： 页面上正在显示的ViewHolder
mChangedScrap：动画过程中的ViewHolder

可直接使用，不需 `onCreateViewHoler()`、 `onBindViewHoler()`

#### 第二级

```
final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
```

默认存储2个刚移出屏幕的ViewHolder, 可直接使用，不需 `onCreateViewHoler()`、 `onBindViewHoler()`


#### 第三级

```
private ViewCacheExtension mViewCacheExtension;
```

用户自定义缓存，不推荐

#### 第四级


```
RecycledViewPool mRecyclerPool;
```

默认存储5个移出屏幕的ViewHolder，复用无需重新调用 `onCreateViewHoler()`, 要重新执行 `onBindViewHoler()`。

<br><br><br><br>
