---
title: "Typescript 中的类型动态约束"
date: 2024-01-04T16:25:14-05:00
lastmod: 2024-01-04T16:25:14-05:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "编写静态的类型大家都会，来看看如何给动态的类型进行约束"
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

# 需求

现在我需要定义一个类型，它要根据不同的 `enum` 来约束其中的参数。

比如要开发一个 `function` 它的功能是获取全平台的 API 接口实例，但是由于不同平台的 API 接口不一致，有些平台需要额外传递参数，因此期望这个 `fucntion` 的调用方式如下;

```typescript
//  调用 macos 的接口
getAPIInstance({
  platform: PlatformEnum.MACOS,
  platformConfig: {
    token: "token_string",
  },
});

//  调用 window 的接口
getAPIInstance({
  platform: PlatformEnum.WINDOW,
  platformConfig: {
    version: "1.2.0",
  },
});

//  调用 android 的接口，这个不需要传递额外的参数
getAPIInstance({
  platform: PlatformEnum.ANDROID,
});
```

可以注意到每个平台需要传递的 `platformConfig` 都不一样，有些平台甚至不需要传递。因此要做同意的类型约束除非使用 `泛型`

```typescript
getAPIInstance<PlatformEnum.WINDOW>({
  platform: PlatformEnum.WINDOW,
  platformConfig: {
    version: "1.2.0",
  },
});
```

这样虽然能解决问题，但是有点不是很优雅，我们希望根据传入的 `platform` 参数能自动推导出 `platformConfig` 的类型，下面看下如何实现。

```typescript
export enum PlatformEnum {
  MACOS = "MACOS",
  WINDOW = "WINDOW",
  LINUX = "LINUX",
  ANDROID = "ANDROID",
  IPHONE = "IPHONE",
  CHROME_BOOK = "CHROME_BOOK",
}

type PlatformConfigDataMap = {
  [PlatformEnum.MACOS]: {
    token: string;
  };
  [PlatformEnum.WINDOW]: {
    version: number;
  };
  [PlatformEnum.LINUX]: {
    publisher: {
      name: string;
      link: string;
    };
  };
  [PlatformEnum.ANDROID]: undefined;
  [PlatformEnum.IPHONE]: {
    token: string;
  };
  [PlatformEnum.CHROME_BOOK]: undefined;
};

export type PlatformConfigType = {
  [K in keyof PlatformConfigDataMap]: PlatformConfigDataMap[K] extends undefined
    ? {
        platform: K;
        platformConfig?: PlatformConfigDataMap[K];
      }
    : { platform: K; platformConfig: PlatformConfigDataMap[K] };
}[keyof PlatformConfigDataMap];
```

测试

```typescript
//  typescript 到这里会报错，提示需要下面的类型
//  { platform: PlatformEnum.MACOS; platformConfig: { token: string; };
getAPIInstance({
  platform: PlatformEnum.MACOS,
});

//  没有错误
getAPIInstance({
  platform: PlatformEnum.ANDROID,
});
```
