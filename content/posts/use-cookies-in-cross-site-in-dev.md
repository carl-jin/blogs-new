---
title: "在本地开发环境中，使用 Cookies 请求跨域接口"
date: 2025-09-02T19:51:24-04:00
lastmod: 2025-09-02T19:51:24-04:00
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

# 背景

需要在本地开发环境中（localhost），调用线上接口（跨域），但是有个关键点，本地 localhost 无法模拟线上登入（因为线上登入是 wordpress 的一个页面，而本地是纯前端项目）

# 解决过程

## 1. 现在线上完成登入

在线上完成登入，确保 cookeis 存储成功

## 2. 本地将接口代理下

将本地接口请求全部改成 `/api` 这样会请求 `locahost/api` 然后在通过下方代理配置

```js

 proxy: {
    "/api": {
        target: "线上接口地址", // 目标后端接口
        changeOrigin: true, // 修改 Origin 头为目标 URL
        secure: true // 如果是 https 接口，且自签名证书，需要设成 false
    }
},
```

转发到线上，这样做的目的是让本地的 cookeis 发送过去。因为浏览器禁止跨域发送 cookies

## 3. 手动复制 cookeis

在线上网站打开 Devtools 中的 Application，然后在 cookies 中找到线上网站设置的所有条目（domain 为线上网站的）

然后切换到本地开发环境运行的网站，在 Devtools 的 Application 添加对应的条目

