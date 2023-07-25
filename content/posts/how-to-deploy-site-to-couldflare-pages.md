---
title: "如何使用 API token 将你的文件部署到 Cloudflare Pages 上"
date: 2023-07-25T16:43:19-04:00
lastmod: 2023-07-25T16:43:19-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "通过基本的 Wrangler 命令将文件部署到 Cloudflare Pages 上，想必大家都知道了，本文将介绍如何使用 API token 的方式将其部署到 Cloudflare Pages 上."
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

## 注意

本篇文章不是面对初学者，继续阅读代表着你已经满足以下条件

1. 有一个 Cloudflare 的账号
2. 安装了 nodejs 和 wrangler

# 创建一个 Page

以下是操作步骤

1. 访问 [dashboard](https://dash.cloudflare.com/)
2. 在左侧侧边栏点击 `Workers & Pages`
3. 在 Overview 中点击那个特别显眼的蓝色按钮 **Create Application**
4. 在 Tabs 切换中选中 `Pages`
5. 动到最下方的 `Create ssing direct upload` 点击 **Upload Assets**
6. 在 project name 中，给项目取一个名字, 然后点击 `Create project`
7. 然后在 `Upload your project assets` 下方的 Dropzone 上拖拽任何的目录到里面（比如创建个目录然后里面只有一个 index.html)
8. 点击右下角的 `Deploy site`
9. 完成！

# 获取 API token

创建 token 的步骤如下

1. 访问 [Api Tokens](https://dash.cloudflare.com/profile/api-tokens)
2. 在 API Tokens 这个板块中点击 `Create Token`
3. 在 API token templates 中选择 `Edit Cloudflare Workers` 这个模版，然后点击 `Use template`
4. 然后在 `Account Resources` 和 `Zone Resources` 中分别选择你的账号和资源就 ok 了
5. 点击最下面的 `Continue to summary`
6. 点击 `Create Token`
7. 此时 Token 会显示出来，点击 `Copy` 将它保存到你的 KeePass 中以后可能会经常用到

# 使用 wrangler 来部署项目

```shell
# 这是一个通过 API Token 将当前目录下的 dist 目录，部署到 carljin-com 这个项目上的 wrangler 命令
CLOUDFLARE_API_TOKEN="API Token" wrangler pages deploy dist --project-name=carljin-com
```

# 使用 Github Action

下面 yml 文件中有两个项目的 secrets

1. CF_API_TOKEN API Token 字符串
2. PROJECT_NAME 项目名称

```yml
name: Build and Deploy to cloudflare
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: 16
          registry-url: "https://registry.npmjs.org"
      - run: npm install
      - run: npm run build

      - name: Deploy to cloudflare
        uses: demosjarco/wrangler-action-node@v1
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          command: pages deploy dist --project-name=${{ secrets.PROJECT_NAME }}
```
