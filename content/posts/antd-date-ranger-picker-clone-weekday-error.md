---
title: "antd 更新到新版本后，出现 DatePicker 或者 RangePicker 出现 clone.weekday is not a function 错误"
date: 2024-11-06T10:29:49-05:00
lastmod: 2024-11-06T10:29:49-05:00
author: ["CarlJin"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 
description: "antd clone.weekday is not a function 错误处理"
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

# Antd DatePicker presets 参数报错解决方案

## 问题描述

在将 Ant Design (antd) 升级到最新版本后，使用 DatePicker 组件的 presets 参数时，遇到了以下错误：

```
clone.weekday is not a function
```

这个错误的原因是 dayjs 需要额外加载 weekday 和 localeData 插件才能正常使用这些功能。

## 解决方案

要解决这个问题，需要在项目中引入并启用必要的 dayjs 插件。具体步骤如下：

1. 首先，在项目入口文件（如 `index.js` 或 `App.jsx`）中添加以下导入语句：

```javascript
import dayjs from 'dayjs'
import weekday from "dayjs/plugin/weekday"
import localeData from "dayjs/plugin/localeData"
```

2. 然后，启用这些插件：

```javascript
dayjs.extend(weekday)
dayjs.extend(localeData)
```

3. 现在就可以正常使用 DatePicker 的 presets 参数了。

## 使用示例

```javascript
import { DatePicker } from 'antd';

const presets = [
  { label: '最近7天', value: dayjs().subtract(7, 'd') },
  { label: '最近30天', value: dayjs().subtract(30, 'd') },
  { label: '最近90天', value: dayjs().subtract(90, 'd') }
];

const Demo = () => (
  <DatePicker presets={presets} />
);
```

## 补充说明

- 这个问题通常出现在 Antd 5.x 版本中
- 确保你的项目已经正确安装了 dayjs 依赖
- 这些插件的引入应该在应用初始化时就完成，建议放在项目的入口文件中

## 相关链接

- [Day.js 官方文档](https://day.js.org/docs/en/plugin/plugin)
- [Ant Design DatePicker 组件文档](https://ant.design/components/date-picker-cn)



