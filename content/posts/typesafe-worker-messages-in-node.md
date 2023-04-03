---
title: "让 worker_threads 之间的通讯支持 typescript"
date: 2023-04-03T09:46:16-04:00
lastmod: 2023-04-03T09:46:16-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "worker_threads 之间通讯是没有 typescript 支持的，并且还不支持回调，本文将介绍如何解决这些问题"
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

# 解决方案

我们使用 [node-typesafe-worker-messages
](https://github.com/carl-jin/node-typesafe-worker-messages) 这个库

来实现 typescript 支持和发送消息的 callback

## 安装

```shell
$ pnpm add node-typesafe-worker-messages -D
```

## 使用

我们需要先定义以下消息的类型，因为主进程会发消息给 worker，

worker 也会发消息给主进程 ，因此我们需要定义两个消息类型

> ⚠️ 注意，消息类型必须用 Promise 作为返回值，即使你返回的是 void

```typescript
//  types.ts

//  定义从主进程发送给每个 worker 的消息类型
type ParentMessage = {
  getWorkProcessId(): Promise<number>;
  getWorkTasksList(): Promise<string[]>;
  printDataFromWorker(
    args: number,
    args2: number,
    args3: number
  ): Promise<void>;
};

//  定义从 worker 发送给主进程的消息类型
type WorkerMessage = {
  getMainProcessId(): Promise<number>;
  getUserInfoById(id: string): Promise<string>;
};
```

## 主进程中使用

```typescript
import { TypeSafeWorkerMessagesInMain } from "node-typesafe-worker-messages";
import type { ParentMessage, WorkerMessage } from "./types.ts";

//  这里将之前定义好的两个类型当成泛型传递过去
const worker = new TypeSafeWorkerMessagesInMain<ParentMessage, WorkerMessage>(
  //  这个类接收的参数和返回值跟 worker_threads 是一样的
  //  它只是对 worker_threads 进行了一层包裹
  join(__dirname, "worker.cjs"),
  {
    workerData: { foo: "bar" },
  }
);

//  所有方法都与 worker_threads 一样
//
worker.on("online", () => {
  // 但是我们提供了两个额外的方法 handle 和 send，
  // 定义他们的目的是为了消息传递时的类型约束和 callbakc 实现
  //  handle 是用于处理消息
  //  send 是用于发送消息
  worker.handle("getMainProcessId", () => {
    return Promise.resolve(1234);
  });

  worker.handle("getUserInfoById", () => {
    return Promise.resolve("Carl");
  });

  worker.send("printDataFromWorker", 1, 2, 3);

  worker.send("getWorkTasksList").then((res) => {
    console.log(res, "getWorkTasksList callback");
  });

  //  当然你还可以继续使用 on("message") 来监听消息
  //  记住，TypeSafeWorkerMessagesInMain 只是对 worker_threads 的一层包裹
  worker.on("message", (action) => {
    if (action === "close") {
      worker.terminate();
    }
  });
});
```

## Worker

> 此时你需要手动关闭它

```typescript
import { TypeSafeWorkerMessagesInWorker } from "node-typesafe-worker-messages";
import type { ParentMessage, WorkerMessage } from "./types.ts";

const _: MessagePort = parentPort as MessagePort;
const parent = TypeSafeWorkerMessagesInWorker<ParentMessage, WorkerMessage>(_);

parent.handle("getWorkProcessId", () => {
  return Promise.resolve(123);
});

parent.handle("printDataFromWorker", (...args) => {
  console.log(args, "from worker");
});

parent.handle("getWorkTasksList", () => {
  return Promise.resolve(["1", "2222"]);
});

setTimeout(() => {
  parent.send("getUserInfoById", "44").then((res) => {
    console.log(res, "getUserInfoById callback from parent with callback!");

    parent.postMessage("close");
  });
}, 2000);
```
