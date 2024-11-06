---
title: 如何使用 Playwright 登入 Google 账号
date: 2024-11-06T10:35:48-05:00
lastmod: 2024-11-06T10:35:48-05:00
author: ["CarlJin"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 
description: "如何使用 Playwright 登入 Google 账号，这取决于具体版本和配置"
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

# 使用 Playwright 模拟登录 Google 账号的完整指南

## 环境要求

- Playwright-core 版本: ^1.48.2
- Google Chrome 浏览器 (本地安装)
- MacOS 环境 (本教程基于 MacOS，Windows 用户需要调整相应路径)

## 安装依赖

```bash
npm install playwright-core@1.48.2
```

## 完整代码示例

首先，让我们定义浏览器启动参数：

```typescript
const getBrowserArgs = () => [
  '--disable-blink-features=AutomationControlled',
  '--no-sandbox',
  '--disable-web-security',
  '--disable-infobars',
  '--disable-extensions',
  '--start-maximized',
  '--window-size=1680,930',
  '--disable-features=IsolateOrigins,site-per-process',
  '--disable-notifications',
  '--disable-no-sandbox',
  '--disable-dev-shm-usage',
  '--aggressive-cache-discard',
  '--disable-cache',
  '--disable-application-cache',
  '--disable-offline-load-stale-cache',
  '--disable-gpu-shader-disk-cache',
  '--media-cache-size=0',
  '--disk-cache-size=0',
  '--disable-extensions',
  '--disable-component-extensions-with-background-pages',
  '--disable-default-apps',
  '--mute-audio',
  '--no-default-browser-check',
  '--autoplay-policy=user-gesture-required',
  '--disable-background-timer-throttling',
  '--disable-backgrounding-occluded-windows',
  '--disable-background-networking',
  '--disable-search-engine-choice-screen',
  '--disable-breakpad',
  '--disable-component-update',
  '--disable-domain-reliability',
  '--disable-sync',
  '--disable-translate',
];
```

然后，创建浏览器实例并配置上下文：

```typescript
const { chromium } = require('playwright-core');

async function setupBrowser(args = {}) {
  const browserArgs = getBrowserArgs();
  const browser = await chromium.launch({
    headless: false,
    executablePath: '/Applications/Google Chrome.app/Contents/MacOS/Google Chrome',
    args: browserArgs,
    ...args,
  });

  const context = await browser.newContext({
    //  Google 似乎根据这个来判断是否为机器人
    //  下面这个 userAgent 在 2024年11月05日，依然可用
    userAgent: 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/130.0.0.0 Safari/537.36',
    viewport: { width: 1280, height: 720 },
    deviceScaleFactor: 1,
  });

  return { browser, context };
}
```

完整的登录示例：

```typescript
async function loginToGoogle() {
  const { browser, context } = await setupBrowser();
  try {
    const page = await context.newPage();
    
    // 访问 Google 登录页面
    await page.goto('https://accounts.google.com');
    
    // 输入邮箱
    await page.fill('input[type="email"]', 'your-email@gmail.com');
    await page.click('button:has-text("Next")');
    
    // 等待密码输入框出现
    await page.waitForSelector('input[type="password"]', { timeout: 5000 });
    
    // 输入密码
    await page.fill('input[type="password"]', 'your-password');
    await page.click('button:has-text("Next")');
    
    // 等待登录完成
    await page.waitForNavigation();
    
    console.log('登录成功！');
    
    // 这里可以继续执行其他操作
    
  } catch (error) {
    console.error('登录过程中出现错误:', error);
  }
}
```
