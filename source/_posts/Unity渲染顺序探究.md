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
cover: https://the-black-sun-imgs.pages.dev/Img/Cover.jpg
---
2023年10月7日更新

---

### 前言

前段时间在群里看见一个有关于求问Unity渲染透明物体与不透明物体渲染顺序的问题，有群友提到了深度与渲染顺序有关，就去找了相关资料学习，并在这里记录一下。

### Unity的UGUI层级渲染顺序管理

主要是层级管理可以看下方这一张图片，其中RenderQueue的数值关系到了渲染队列的划分。unity中的渲染队列主要有5个，分别是背景渲染队列，不透明渲染队列，透明度测试渲染队列，透明度混合渲染队列以及特殊物体渲染队列（主要用来渲染同一个物体存在相互遮挡且透明的情况）。其中最主要的不透明与透明渲染队列的划分也就如下图所示，是以2500为分界线。

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

除此之外还有根据深度z轴判断前后关系，y轴判断前后关系，x轴判断前后关系等，在2.5D游戏中人物与环境的遮挡交互会使用到，这里先不做过多的赘述。

### Unity的渲染队列（2023年10月7日更新）

Unity通过渲染队列来进行如下。在unityShader中的SubShader的**`Queue`**标签可以决定模型改归于哪一个渲染队列，索引号越小表示越早被渲染。渲染队列有以下几种：

| 名称 | 队列索引号 | 描述 |
| --- | --- | --- |
| Background | 1000 | 该队列在其他所有队列之前被渲染，通常用来渲染需要被绘制在背景上的物体 |
| Geometry | 2000 | 默认渲染队列，大多数物体使用的队列，不透明物体使用的队列 |
| AlphaTest | 2450（小于2500为完全不透明） | 需要进行透明度测试的物体使用的队列。 |
| Transparent | 3000 | 该队列中的物体会在前两个队列（Geometry与AlphaTest）渲染之后，再根据从后往前的顺序进行渲染，所有使用了透明度混合的物体使用该队列。 |
| Overlay | 4000 | 该队列用来实现叠加效果，任何需要在最后渲染的物体使用这个队列 |