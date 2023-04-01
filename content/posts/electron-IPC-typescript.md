---
title: "Electron 中使用 type-safe 的 IPC"
date: 2023-03-31T23:02:08-04:00
lastmod: 2023-03-31T23:02:08-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "让我们来看看如何在主进程和渲染进程通讯过程中（IPC）实现 typescript 的支持"
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

随着项目越来越大，各种 IPC 之间的数据传输越来越混乱，因此急需 typescript 的支持，以达到完整的参数和返回值的提示，减少来回查阅的成本

# 直接上代码

## 类型文件定义

首先我们定义一个 `types.ts` 文件，用来存放 IPC 中每个消息的类型

```typescript
//  由于我们要实现 主进程 和 渲染进程 的双向通讯，
//  因此定义了俩 type

//  从 渲染进程 传过来的消息类型
export type RenderMessage = {
  getUserNameById(userID: number): Promise<string>;
};

//  从 主进程 发送到渲染进程的消息类型
export type MainMessage = {
  newUserJoin(userID: number): void;
};
```

## 主进程 IPC 实现

来看看主进程中的 `IPC` 类实现

```typescript
import { ipcMain, BrowserWindow } from "electron";

type MessageObj<T> = {
  [K in keyof T]: (...args: any) => void;
};

export class IPCMain<
  MessageType extends MessageObj<MessageType>,
  BackgroundMessageType extends MessageObj<BackgroundMessageType>
> {
  channel: string;
  listeners: Partial<Record<keyof MessageType, any>> = {};

  constructor(channel: string = "IPC-bridge") {
    this.channel = channel;

    this._bindMessage();
  }

  on<T extends keyof MessageType>(
    name: T,
    fn: (...args: Parameters<MessageType[T]>) => ReturnType<MessageType[T]>
  ): void {
    if (this.listeners[name])
      throw new Error(`消息处理器 ${String(name)} 已存在`);
    this.listeners[name] = fn;
  }

  off<T extends keyof MessageType>(action: T): void {
    if (this.listeners[action]) {
      delete this.listeners[action];
    }
  }

  async send<T extends keyof BackgroundMessageType>(
    name: T,
    ...payload: Parameters<BackgroundMessageType[T]>
  ): Promise<void> {
    // 获取所有打开的窗口
    const windows = BrowserWindow.getAllWindows();

    // 向每个窗口发送消息
    windows.forEach((window) => {
      // @ts-ignore
      window.webContents.send(this.channel, {
        name: name,
        payload: payload,
      });
    });
  }

  _bindMessage() {
    ipcMain.handle(this.channel, this._handleReceivingMessage.bind(this));
  }

  async _handleReceivingMessage(
    _: any,
    payload: { name: keyof MessageType; payload: any }
  ) {
    try {
      if (this.listeners[payload.name]) {
        const res = await this.listeners[payload.name](...payload.payload);
        return {
          type: "success",
          result: res,
        };
      } else {
        throw new Error(`未知的 IPC 消息 ${String(payload.name)}`);
      }
    } catch (e: any) {
      return {
        type: "error",
        error: e.toString(),
      };
    }
  }
}
```

## 渲染进程 IPC 实现

我们在渲染进程里面建立以下这个类

```typescript
import { ipcRenderer, on } from "#preload";
import { ErrorMessage } from "@/utils/message";

type MessageObj<T> = {
  [K in keyof T]: (...args: any) => void;
};

export class IPCRenderer<
  MessageType extends MessageObj<MessageType>,
  BackgroundMessageType extends MessageObj<BackgroundMessageType>
> {
  channel: string;
  listeners: Partial<Record<keyof BackgroundMessageType, any>> = {};

  constructor(channel: string = "IPC-bridge") {
    this.channel = channel;

    this._bindMessage();
  }

  send<T extends keyof MessageType>(
    name: T,
    ...payload: Parameters<MessageType[T]>
  ): Promise<Awaited<ReturnType<MessageType[T]>>> {
    return new Promise(async (res, rej) => {
      const data = await ipcRenderer.invoke(this.channel, {
        name: String(name),
        payload,
      });
      if (data.type === "success") {
        return res(data.result);
      } else {
        //  主进程如果返回错误的话，在这里显示到 UI 上
        ErrorMessage(data.error);
        return rej(data.error);
      }
    });
  }

  on<T extends keyof BackgroundMessageType>(
    name: T,
    fn: (...args: Parameters<BackgroundMessageType[T]>) => void
  ): () => void {
    this.listeners[name] = this.listeners[name] || [];

    this.listeners[name].push(fn);

    //  提供删除方法，方便 react 中的 useEffect 使用
    return () => {
      if (this.listeners[name].includes(fn)) {
        const index = this.listeners[name].indexOf(fn);
        this.listeners[name].splice(index, 1);
      }
    };
  }

  _handleReceivingMessage(
    _,
    payloadData: { name: keyof BackgroundMessageType; payload: any }
  ) {
    const { name, payload } = payloadData;

    if (this.listeners[name]) {
      for (let fn of this.listeners[String(name)]) {
        fn(...payload);
      }
    }
  }

  _bindMessage() {
    on(this.channel, this._handleReceivingMessage.bind(this));
  }
}
```

# 使用

## 主进程

```typescript
//  首先 new 一下
const IPC = new IPCMain<RenderMessage, MainMessage>();

//  绑定处理函数
//  完全的 typescript 检测！
IPC.on("getUserNameById", (userID: string) => {
  return Promise.resolve("张三");
});

//  主进程向渲染进程发消息
//  完全的 typescript 检测！
IPC.send("newUserJoin", 9);
```

## 渲染进程

```tsx
const IPC = new IPCRenderer<RenderMessage, MainMessage>();

function App() {
  useEffect(() => {
    //  完全的 typescript 检测
    const remove = IPC.on("newUserJoin", (id) => {
      console.log(id);
    });

    return () => {
      remove();
    };
  }, []);

  function handleGetName() {
    //  完全的 typescript 检测！
    IPC.send("getUserNameById", 9).then((userName) => {
      console.log(userName);
    });
  }

  return <button onClick={handleGetName}>获取名称</button>;
}
```
