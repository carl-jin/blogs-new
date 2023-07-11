---
title: "vite 如何在 package.json 中导出多个模块"
date: 2023-07-08T20:30:17-04:00
lastmod: 2023-07-08T20:30:17-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "我们已经很熟悉在使用 vite 导出单个模块时的 package.json 写法，本文将为你介绍如何处理导出多个模块的情况"
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

假设我们要开发一个工具库，导出模块 A，B，C 现在希望他们能达到最大程度的 Tree shake，因此我们决定导出多个文件。

# 写个打包脚本

```typescript
//  scripts/build.ts

import { build } from "vite";
import { dirname, resolve } from "path";
import { fileURLToPath } from "url";
import terser from "@rollup/plugin-terser";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
export const r = (...args: string[]) =>
  resolve(__dirname, "..", ...args).replace(/\\/g, "/");

//  在这里定义你需要打包的模块
const files = ["A.ts", "B.ts", "C.ts"];

for (let fileName of files) {
  const filePath = r(`src/${fileName}`);
  const name = fileName.split(".")[0];

  build({
    root: r("src"),
    build: {
      outDir: r("dist"),
      emptyOutDir: false,
      sourcemap: false,
      cssCodeSplit: false,
      lib: {
        entry: filePath,
        name: name,
        formats: ["es"],
      },
      rollupOptions: {
        output: {
          entryFileNames: `${name}.js`,
        },
        plugins: [terser()],
      },
    },
  });
}
```

# 配置 package.json

```json
{
  "name": "mymodules",
  "private": false,
  "version": "0.0.1",
  "type": "module",
  "main": "./dist/A.js",
  "module": "./dist/A.js",
  "types": "./dist/A.d.ts",
  "exports": {
    ".": "./dist/A.js",
    "./A": "./dist/A.js",
    "./B": "./dist/B.js",
    "./C": "./dist/C.js"
  },
  "typesVersions": {
    "*": {
      "A": ["./dist/A.d.ts"],
      "B": ["./dist/B.d.ts"],
      "C": ["./dist/C.d.ts"]
    }
  },
  "files": ["dist"],
  "scripts": {
    "build": "run-s build:*",
    "build:clear": "rimraf dist",
    "build:js": "esno scripts/build.ts",
    "build:type": "tsc -p ./tsconfig.json -emitDeclarationOnly"
  }
}
```
