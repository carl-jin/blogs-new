---
title: "如何在 Electron 的 Main 进程中使用 console 将信息打印到渲染进程"
date: 2024-07-10T15:30:26-04:00
lastmod: 2024-07-10T15:30:26-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "几行代码搞定"
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

```typescript
//  在 Main 代码中编写一个 log 方法
export function log(...data) {
  if (!mainWindow) return;
  mainWindow.webContents.executeJavaScript(
    `  try {
          console.log('%cFROM MAIN', 'color: #800', JSON.parse(${JSON.stringify(
            data
          )}));
        } catch (e) {
          console.log('%cFROM MAIN', 'color: #800', ${JSON.stringify(data)});
        }`
  );
}
```

```typescript
//  mainWindow 来自于创建 BrowserWindow 的返回值
const mainWindow = new BrowserWindow({
  //  ....
});
```
