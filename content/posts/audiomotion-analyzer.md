---
title: "使用 audiomotion-analyzer 实现音频可视化"
date: 2023-12-20T22:45:01-05:00
lastmod: 2023-12-20T22:45:01-05:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "本文将介绍如何使用 audiomotion-analyzer 实现音频可视化"
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

![音频可视化](https://i.imgur.com/RXB8Zkk.png)

## 安装

```shell
pnpm install audiomotion-analyzer
```

## 使用

```typescript
import AudioMotionAnalyzer from "audiomotion-analyzer";

let micStream: any = null;

//  以下配置上方参考图的效果
const audioMotion = new AudioMotionAnalyzer(audioMotionRef.value, {
  mode: 10,
  bgAlpha: 0.7,
  fillAlpha: 0.6,
  gradient: "classic",
  lineWidth: 2,
  lumiBars: false,
  maxFreq: 16000,
  radial: false,
  reflexAlpha: 1,
  reflexBright: 1,
  reflexRatio: 0.5,
  showBgColor: false,
  showPeaks: false,
  showScaleX: false,
  overlay: true,
  height: 100,
});

//  获取音频的 mediaStream
navigator.mediaDevices
  .getUserMedia({
    audio: true,
    video: false,
  })
  .then((stream) => {
    currentAudioDeviceId.value = props.audioDeviceId;

    micStream && micStream?.disconnect?.();
    //  更新音频输入源
    micStream = audioMotion.audioCtx.createMediaStreamSource(stream);
    audioMotion.connectInput(micStream);

    audioMotion.volume = 0;
  })
  .catch((err) => {
    console.error(err);
    Modal.error({
      title: "错误",
      content: "用户拒绝了音频输入权限请求",
    });
  });
```

## 更多配置
[https://github.com/hvianna/audioMotion-analyzer](https://github.com/hvianna/audioMotion-analyzer)
