---
title: "Swiper.js 使用 freeMode 后出现 slide 间距异常问题"
date: 2023-08-24T11:34:47-04:00
lastmod: 2023-08-24T11:34:47-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "Swiper.js 使用 freeMode 后出现 slide 间距异常问题"
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

# Swiper 配置

```typescript
new window["Swiper"](`#${randomID}`, {
  spaceBetween: 48,
  freeMode: true,
  direction: "horizontal",
  effect: "slide",
  pagination: false,
  slidesPerView: "auto",
  autoplay: false,
  slidesOffsetAfter: 48,
  slidesOffsetBefore: 48,
});
```

# 实际效果

![img](https://i.imgur.com/wnsp7Yx.png)

可以看到两张图片之间空出了很大的距离，即使给 slide 设置 固定的 width 也不行。

# 解决方法

添加以下样式

```css
.swiper-slide {
  flex-shrink: unset;
}
```
