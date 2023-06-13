---
title: "Vite 热更新失效"
date: 2023-06-12T23:05:28-04:00
lastmod: 2023-06-12T23:05:28-04:00
author: ["CarlJin"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 
description: "Vite 热更新失效，报错 WebSocket connection to 'ws://localhost/' failed: "
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


## 报错
今天更新了 vite 到 4.3.0 以上后，打开页面出现控制台报错导致热更新失效
```text
WebSocket connection to 'ws://localhost/' failed:
Uncaught (in promise) DOMException: Failed to construct 'WebSocket': The URL 'ws://localhost:undefined/' is invalid.  
```


## 解决方法

将下面的配置添加到 vite.config.js
```text
server: {
    port: 5173,
    strictPort: true,
    hmr: {
      port: 5173,
    },
  },
```

