---
title: "如何解决 nodejs 中的 child_precess.fork 在 Electron 项目中监听不到退出信息问题"
date: 2023-07-25T17:13:18-04:00
lastmod: 2023-07-25T17:13:18-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "在 Electron 想要执行 node 脚本并且监听退出时间，但是无论是 exit 或者 close 都不能按预期工作，本文将介绍如何解决"
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

# 背景

我们现在要在 Electron 项目中通过 node 执行一个 js 文件。目前比较合适的方法是使用 fork, 但是却无法监听退出时间。

```typescript
// foo.js
console.log("bar");
```

```typescript
// main process
import { fork } from "node:child_process";

const child = fork("foo.js");

child.on("exit", () => {
  console.log("根本不执行");
});
child.on("close", () => {
  console.log("根本不执行");
});
```

# 解决方法

使用 [utilityProcess](https://www.electronjs.org/docs/latest/api/utility-process)

```typescript
//  foo.js
console.log("bar");
process.parentPort.postMessage("exitManually");
process.exit(0);
```

```typescript
//  main process
import { utilityProcess } from "electron";

const child = utilityProcess.fork("foo.js");
child.on("message", (msg) => {
  if (msg === "exitManually") {
    console.log("执行完了！");
  }
});
```
