---
title: "如何在 Vite 中使用 JavaScript Obfuscator Tool"
date: 2023-07-25T17:06:47-04:00
lastmod: 2023-07-25T17:06:47-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "有些项目我们可能不想让使用者得到源代码，虽然 JS 在 Devtools 中查看并且调试，但是经过 Obfuscator 后的代码，将大大增加阅读和调试的时间。本文将介绍如何在 Vite 中使用 JavaScript Obfuscator "
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

# 前言

首先感谢 [JavaScript Obfuscator Tool](https://obfuscator.io/) 提供的这个工具。

# 直接上代码

这里我们写个 vite 插件，直接调用即可

```typescript
import JavaScriptObfuscator from "javascript-obfuscator";
import { minify } from "terser";

export function codeObfuscator(): any {
  return [
    {
      name: "code-obfuscator",
      enforce: "post",
      async transform(code) {
        //  先压缩然后再进行混淆
        //  因为 import.meta.env 可能会出现引用对象的问题
        //  比如 {a:'foo',b:'bar'}.a
        //  压缩后 'foo'
        const res = await minify(code);
        const content = res.code || code;

        //  在下面配置你的参数
        let obfuscationResult = JavaScriptObfuscator.obfuscate(content, {
          compact: true,
          controlFlowFlattening: true,
          controlFlowFlatteningThreshold: 1,
          numbersToExpressions: true,
          simplify: true,

          stringArray: true,
          stringArrayCallsTransform: true,
          stringArrayEncoding: ["base64", "rc4"],
          stringArrayIndexShift: true,
          stringArrayRotate: true,
          stringArrayShuffle: true,
          stringArrayWrappersCount: 3,
          stringArrayWrappersChainedCalls: true,
          stringArrayWrappersParametersMaxCount: 2,
          stringArrayWrappersType: "variable",
          stringArrayThreshold: 1,

          splitStrings: true,
          unicodeEscapeSequence: true,
        });

        return obfuscationResult.getObfuscatedCode();
      },
    },
  ];
}
```
