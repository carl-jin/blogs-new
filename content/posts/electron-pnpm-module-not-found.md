---
title: "Electron Pnpm Module Not Found"
date: 2023-03-28T15:46:18-04:00
lastmod: 2023-03-28T15:46:18-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  - electron
description: "记一次使用 pnpm 打包 electron 项目时，遇到的 module 找不到问题"
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

使用 pnpm 打包 electron 后，运行时报错，提示某 "tslib" 找不到，经过查找这个库是被 `typeorm` 这个库依赖于`dependencies`当中，我们项目中把 `typeorm` 也放到了 `dependencies` 当中

那么按理说打包时候，typeorm 的 dependencies 依赖也会下载下来，并且放到 node_modules 里面。但是事实上并没有

经过查找后发现是 electron-builder 对 pnpm 支持不好的问题，
[github issue](https://github.com/electron-userland/electron-builder/issues/6289#issuecomment-1042620422)
只需要在 `.npmrc` 中添加下面一行就行了

```text
node-linker=hoisted
```

⚠️，这样就会丢失 pnpm 的优势，也就是软连接的 module 引用
