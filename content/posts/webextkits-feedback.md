---
title: "Webextkits 使用体验报告"
date: 2023-11-02T08:03:26-04:00
lastmod: 2023-11-02T08:03:26-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: ""
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

## 什么是 Webextkits

[Webextkits](https://webextkits-docs.pages.dev/) 是一个浏览器扩展开发模版，旨在统一开发标准。它提供对通用功能的支持，让开发者能够专注于功能实现，从而加快开发进程，并避免后期维护中的问题。

## 它带来的便利

### InjectScript

`Webextkits` 提出了 `InjectScript` 的概念，是指的通过 background.js 中注入到页面的脚本文件, 如下面 injects/index.js 在符合 https://github.com/* 规则的网页加载完成后被注入到页面当中。

```typescript
browser.scripting.registerContentScripts([
  {
    id: `inject-module`,
    js: ["injects/index.js"],
    matches: ["https://github.com/*"],
    runAt: "document_end",
    world: "MAIN",
  },
]);
```

使用 InjectScript 而不是 contentScript 的主要原因是在与 MV3 中强制对 contentScript 实行了孤立政策, 见官方文档 Work in isolated worlds

> Content scripts live in an isolated world, allowing a content script to make changes to its JavaScript environment without conflicting with the page or other extensions' content scripts.

这意味着我们无法在 contentScript 中读取或者修改页面中 window 上的数据.

为了让我们的扩展能继续像 MV2 一样能够读取并且修改页面中 window 上的数据，我们需要通过 `browser.scripting.registerContentScripts` 这个 api 来注入代码文件到页面中，进而达到获取和修改页面中 window 上的数据。

这种通过 `browser.scripting.registerContentScripts` api 注册的脚本文件，称之为 InjectScript

配合 `Webextkits` 提供的 vite 插件内置的 `InjectScripts` 的转换和 `externals` 提取，这对于我来说减少了不少项目初始化时间，何况安装这个模版只需要一行命令

```shell
$ pnpm create webextkits@latest
```

### 消息管理的 typescript 支持

在浏览器开发中，`background`, `content`, `options` 和 `popup` 之间进行消息传递本身是没有 `typescript` 支持的，一旦消息数量变得繁杂，对后序的开发和维护都是一件头疼的事情.

何况我们还需要在 `injectScript` 中传递消息到 `background` 对于没有 `typescript` 支持的消息系统, 这很头疼。

`Webextkits` 的解决方案是通过 `@webextkits/messages-center` 这个库来处理不同消息之间的传递，它简单的就像 jquery 事件监听一样。

```typescript
const mc = new MessageInstance<InjectMessageType, BackgroundMessageType>(
  extId,
  true
);

//  在这里你可以对某个消息进行监听处理
mc.on("readUserName", async () => {
  const user = await getBucket("user");
  return user.name;
});
```

就这么简单，而且还具备完整的 typescript 支持，这对于记忆力不好的我来说，再也不用花时间找事件名称和参数定义了。

### storage.local 的 typescript 支持

> 无论简单还是复杂的扩展，一旦涉及到 `storage.local` 的调用，如果没有 `typescript` 支持，难免的因着时间的流逝，维护成本也会逐步的增加（特别是非原作者维护时），让存储的数据支持 typesafe 这是必然的结果。

说说个人感受，在 storage.local 中存数据，如果没有想 ajv 这样的库做抽象（ajv 不支持在 background 运行），对于已经习惯 typescript 带来的便利性的我，真的是有点难以接受，好在 `Webextkits` 也对 storage.local 的操作进行了封装。

#### 定义 schema

```typescript
//  user.ts
import { JSONSchemaType } from "@webextkits/storage-local";

export type UserSchemaType = {
  name: string;
  age: number;
};

export const UserSchema: JSONSchemaType<UserSchemaType> = {
  type: "object",
  properties: {
    name: {
      type: "string",
      default: "default name",
    },
    age: {
      type: "number",
      default: 18,
    },
  },
  default: {},
  required: ["name", "age"],
};
```

#### 使用

```typescript
import { schema, SchemaType } from "@/schema/index";

//  使用 useStorageLocal 这个 hook
const { updateBucket, setBucket, getBucket, deleteBucket } =
  //  必须传入 schema 和 SchemaType 这个泛型，这样才会有代码提示
  useStorageLocal<SchemaType>(schema);

//  更新 user 这个 storage 的数据
updateBucket("user", (bucket) => {
  bucket.name = name;
  return bucket;
});
```

有兴趣的可以安装试试
