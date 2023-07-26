---
title: "CRXJS 中如何使用 dotenv"
date: 2023-07-25T20:25:59-04:00
lastmod: 2023-07-25T20:25:59-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "CRXJS 是一个浏览器扩展开发模版，基于 vite 实现了热更新和文件管理。但是如果直接通过 vite 配置 dotenv，始终无法正常通过 import.meta.env 读取内容，本文将介绍如何解决"
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

# 直接上代码

```typescript
//  vite 插件
import dotenv from "dotenv";
import { r } from "../scripts/utils";
import replace from "@rollup/plugin-replace";

const envData = dotenv.config({ path: r(".env") }).parsed ?? {};

export function defineEnv(): any {
  return replace({
    ...Object.entries(envData).reduce((prev, current) => {
      prev[`import.meta.dotenv.${current[0]}`] = `"${current[1]}"`;
      return prev;
    }, {}),
    preventAssignment: true,
  });
}
```
