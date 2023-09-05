---
title: Unity渲染顺序探究
date: 2023-09-05 20:26:33
tags: [Unity,性能优化,图形渲染]
categories: Unity
description: Unity的渲染顺序探究
copyright_author: 墨墨辰
copyright_author_href: https://the-black-sun.github.io/
copyright_url: https://the-black-sun.github.io/2023/09/05/Unity渲染顺序探究/
copyright_info: 此文章版权若没有特别注明，则归墨墨辰所有，如有转载，请注明来自原作者。
cover: https://i0.hdslb.com/bfs/article/83916cd63e6210d2593f3c0fb12661542115cf7f.jpg
---
### 前言

前段时间在群里看见一个有关于求问Unity渲染透明物体与不透明物体渲染顺序的问题，有群友提到了深度与渲染顺序有关，就去找了相关资料学习，并在这里记录一下。

### Unity的UGUI层级渲染顺序管理

![Untitled](../Unity渲染顺序探究/Untitled.png)

摄像机的Depth：值越大，渲染物体越靠上，**摄像机会根据Depth从小到大的顺序，渲染各自Culling Mask的层。**

RenderQueue：渲染物体的透明度，小于2500的先渲染

SortingLayer：SortingLayer在Inspector面板中点击Tag -> AddTag -> SortingLayer，可以添加自定义的sortingLayer，默认的sortingLayer为Default

Order In Layer：SortingLayer中的内部渲染排序。

![Untitled](../Unity渲染顺序探究/Untitled1.png)

![Untitled](../Unity渲染顺序探究/Untitled2.png)

RectTransform.SetSiblingIndex：设置显示与渲染顺序，值越大，越靠上

- SetAsFirstSibling：移动到所有兄弟节点的第一个位置（Hierarchy同级最上面，先渲染，显示在最下面）
- SetAsLastSibling：移动到所有兄弟节点的最后一个位置（Hierarchy同级最下面，后渲染，显示在最上面）
- GetSiblingIndex：获得该元素在当前兄弟节点层级的位置
- SetSiblingIndex：设置该元素在当前兄弟节点层级的位置