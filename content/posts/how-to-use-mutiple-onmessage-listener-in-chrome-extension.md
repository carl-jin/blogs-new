---
title: "如何在 Chrome Extension 中使用多个 onMessage 的 Listener"
date: 2023-07-26T20:51:39-04:00
lastmod: 2023-07-26T20:51:39-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "当我们的扩展变得复杂时，往往需要进行拆分这时就可能遇到需要将 onMessage 的处理拆分成多个 Listener 处理，本文将介绍如何实现"
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

# 原则

如官网说的

> If multiple pages are listening for onMessage events, only the first to call sendResponse() for a particular event will succeed in sending the response. All other responses to that event will be ignored.

> For new extensions you should prefer promises over callbacks. If you're using callbacks, the sendResponse() callback is only valid if used synchronously, or if the event handler returns true to indicate that it will respond asynchronously. The sendMessage() function's callback will be invoked automatically if no handlers return true or if the sendResponse() callback is garbage-collected.

多个 Listener 之间如何确定该返回那个 Listener 的执行结果，是通过谁先调用 sendResponse 并且有返回值判断的，这也就意味着我们需要保证每个 Listener 未处理逻辑是，需要保证没有返回值，比如下面这种.

```typescript
browser.runtime.onMessage.addListener((payload, sender, res) => {
  if (payload.type === "foo") {
    return "bar";
  }
});
```

# 注意

如果你的 Listener 需要处理异步逻辑，**请不要**按以下写.

```typescript
//  这是一个错误的例子，如果我们的 listener 是 async function 那么默认就有一个返回值是 Promise<void>
browser.runtime.onMessage.addListener(async (payload) => {
  if (payload.type === "reload") {
    return await reload();
  }
});
```

下面这个才是正确的写法

```typescript
browser.runtime.onMessage.addListener((payload, sender, sendResponse) => {
  if (payload.type === "reload") {
    reload().then((res) => {
      sendResponse(res);
    });
    return true;
  }
});
```

# 直接上代码

实现起来很简单，就跟 DOM 事件监听一样，只需要写多个监听器就行了

```typescript
//  background.js
browser.runtime.onMessage.addListener((payload, sender, res) => {
  if (payload.id === "foo") {
    setTimeout(() => {
      res("bar");
    }, 1000);
    return true;
  }
});

browser.runtime.onMessage.addListener((payload) => {
  if (payload.type === "reload") {
    return reload();
  }
});
```
