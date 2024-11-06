---
title: Chrome 浏览器使用 crxjs 开发时如何解决 CSP 错误
date: 2024-11-06T12:50:31-05:00
lastmod: 2024-11-06T12:50:31-05:00
author: ["CarlJin"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 
description: 当我们用 crxjs 开发 chrome 插件时，在执行 content.js 或者使用 scripting.registerContentScripts 来注入代码到页面中时遇到 csp 问题如何处理

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
这个问题是由于 chrome@v130 版本更新后， chrome.runtime.getFile 返回的不再是 chromeid 的链接，而是一个 uuid4 的链接，因此导致了 csp 问题，让我们来看下解决方法

## 1. 更新 `@crxjs/vite-plugin`
更新 `@crxjs/vite-plugin`到最新的 `2.0.0-beta.28` 版本

## 2. 添加一个 `declarativeNetRequest` 规则来禁止 CSP 头信息

```typescript
//  useCSP.ts
type Rule = any;

export function useCSP(hostMatch: string[], additionalRules?: Rule[]) {
  (async () => {
    let currentIndex = 0;
    let r = await chrome.declarativeNetRequest.getDynamicRules(),
      n = r.map((e) => e.id);

    const rules = hostMatch.map((domain, index) => {
      currentIndex = index + 1;
      return {
        id: currentIndex,
        priority: 1,
        action: {
          type: "modifyHeaders",
          responseHeaders: [
            { header: "Content-Security-Policy", operation: "remove" },
            {
              header: "Content-Security-Policy-Report-Only",
              operation: "remove",
            },
          ],
        },
        condition: {
          regexFilter: domain,
          resourceTypes: ["main_frame", "xmlhttprequest"],
        },
      } as Rule;
    });
    console.log(rules);

    if (additionalRules && additionalRules.length > 0) {
      additionalRules.map((rule) => {
        rule.id = ++currentIndex;
        rules.push(rule);
      });
    }

    await chrome.declarativeNetRequest.updateDynamicRules({
      removeRuleIds: n,
      addRules: rules as any,
    });
  })();
}
```

在 background.js 中使用它
```typescript
//  background.js
useCSP(['https://google.com/*']);
```

在 manifest.json 中添加 `declarativeNetRequest` 权限
```json
{
  "permissions": [
    "declarativeNetRequest"
  ]
}
```

