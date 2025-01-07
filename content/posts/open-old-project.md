---
title: "记录一次打开老项目报错的处理方法"
date: 2025-01-06T21:57:12-05:00
lastmod: 2025-01-06T21:57:12-05:00
author: ["CarlJin"]
keywords: 
- 
categories: 打开老项目报错记录，新电脑是 M1 芯片，打开之前的 node 10.16.3 项目
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

# 一 遇到使用 nvm 无法切换 node 版本问题

使用 nvm 切换 node 版本时，不管怎么切换，node 版本都是 system 版本，无法切换到 16.10.3（即使 nvm 安装了 16.10.3 版本，并且已经提示切换到了 16.10.3 版本）。

## 原因

由于电脑 node 是直接从 brew 安装的，而不是通过 nvm 进行安装的，导致没办法切换

## 解决方法

首先卸载本地的 node

```shell
# 卸载本地 node
brew uninstall node
# 通过 brew 安装 nvm，如果安装了 nvm，则跳过
brew install nvm
# 通过 nvm 安装 node 16.10.3
nvm install 16.10.3
# 切换到 node 16.10.3
nvm use 16.10.3
```

# 二 安装时，出现 gyp 错误，提示找不到 `python`

## 原因

由于 node 10.16.3 项目依赖的 python 版本是 2.7.18，而新电脑的 python 版本是 3.12.0，导致出现 gyp 错误。

## 解决方法

安装 python 2.7.18

```shell
brew install python@2
```

# 三 即使安装了 python 2.7.18，仍然报错

## 原因

node-gyp 始终没办法编译 node-sass@4.14.1，导致报错。

## 解决方法

在 package.json 中 删除 node-sass@4.14.1 的依赖，然后重新安装依赖
