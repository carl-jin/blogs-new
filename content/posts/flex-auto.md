---
title: "CSS 中如何实现文本宽度在容器中自适应同时满足 ellipsis"
date: 2023-12-31T10:09:40-05:00
lastmod: 2023-12-31T10:09:40-05:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "使用 ellipsis 时一般都需要指定文本宽度，不指定的话同 row 的其它内容可能会被挤走。让我们来看下如何让文本占用剩余 row 的空间并且具备 ellipsis"
weight:
slug: ""
draft: false # 是否为草稿
comments: true # 本页面是否显示评论
reward: false # 打赏
mermaid: true #是否开启mermaid
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示路径
cover:
  image: "" #图片路径例如：posts/tech/123/123.png
  zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
  caption: "" #图片底部描述
  alt: ""
  relative: false
---

# 场景

让我们来下面这个布局

```html
<div style="display: flex;flex-direction: row;justify-content: space-between">
  <div style="text-overflow: ellipsis;overflow: hidden;white-space: nowrap">
    这是用户输入的长文本，它可能会很长
  </div>
  <button>清空</button>
</div>
```

如果我们不指定`文本内容`的宽度，会导致用户输入过长时，`清空`按钮直接被挤走。

# 解决方法

添加 `flex: 1 1 auto`

```html
<div
  style="text-overflow: ellipsis;overflow: hidden;white-space: nowrap;flex: 1 1 auto"
>
  这是用户输入的长文本，它可能会很长
</div>
```

这里的意思是 表示一个用于 Flexbox 布局的快捷属性。它相当于以下属性的缩写：

```text
flex-grow: 1;
flex-shrink: 1;
flex-basis: auto;
```

1. flex-grow 定义了项目在分配多余空间时的放大能力。
2. flex-shrink 定义了项目在空间不足时的缩小能力。
3. flex-basis 定义了项目在未伸缩之前的初始大小。
   设置 `flex: 1 1 auto;` 会使项目等比例地占据剩余空间，允许收缩，并以内容的自然宽度作为基准大小。
