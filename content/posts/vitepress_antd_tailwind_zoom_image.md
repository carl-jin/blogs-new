---
title: "VitePress + Tailwind + ant-design-vue + 图片放大 功能使用"
date: 2023-12-11T09:21:43-05:00
lastmod: 2023-12-11T09:21:43-05:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "Vitepress 作为一个简单、强大和快速的现代 SSG 框架，除了写 Markdown 之外还提供 Vue 模版的渲染，本文将结合例子演示如何在 vitepress 中使用 Tailwind、ant-design-vue 和 图片放大 功能"
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

# Tailwind 使用

## 1. 首先安装 tailwind 依赖

```shell
$ pnpm add tailwindcss @tailwindcss/postcss7-compat postcss autoprefixer -D
```

## 2. 生成 tailwind 配置文件

在根目录下执行

```shell
pnpm dlx tailwindcss init
```

执行完上面的命令后会在根目录生成一个 `tailwind.config.js`

## 3. 配置 `tailwind.config.js`

> 注意下面的配置是假设 vitepress 的入口文件夹为 ./docs

修改 `tailwind.config.js` 配置如下

```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "./docs/**/*.js",
    "./docs/**/*.vue",
    "./docs/**/*.ts",
    "./docs/**/*.md",
  ],
  options: {
    safelist: ["html", "body"],
  },
};
```

## 4. 配置 Tailwind 引用 css

新建 `docs/.vitepress/theme/tailwind.css`

内容如下

```css
@tailwind base;

@tailwind components;

@tailwind utilities;
```

然后在 `docs/.vitepress/theme/index.ts` 文件中 `import` 这个 css 文件即可

```javascript
//  docs/.vitepress/theme/index.ts
import "./tailwind.css";

/** 其它代码略过 **/
```

## 5. 创建 postcss.config.js

在根目录下创建 `postcss.config.js` 内容如下

```javascript
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};
```

现在已经可以在项目中正常的使用 tailwind 了

```markdown
::: tip tailwind 演示
下面是一个 tailwind 演示，这是一个 markdown 文档
:::

<span class="text-2xl">2xl 字体</span>
```

# ant-design-vue 使用

VitePress 本身就支持 vue 模版的渲染，也就是你可以直接在 markdown 中写 vue 代码，请参考官方文档

[https://vitepress.dev/guide/using-vue](https://vitepress.dev/guide/using-vue)

这里介绍的是如何使用 `ant-design-vue` 并且如何设置 `ConfigProvider`

## 1. 创建 vue 文件夹

官方虽然提供 `vue` 模版的渲染，但是直接在 markdown 里面写所有的 vue 逻辑，这似乎有点难以接受（主要是 webstorm 不提供支持）

因此尽量将 vue 逻辑抽离出来，放到 `.vue` 的文件夹中，markdown 里面只处理简单的引用。

所以创建 `docs/vue` 作为 vue 文件的存放目录

下面以创建一个分享链接，并且在每个页面底部显示的功能作为例子

## 2. 创建一个公用的 ConfigProvider 组件

因为 vitepress 本身自带 dark 和 light 主题切换，如果我们不为组件提供 `ConfigProvider` 那么 `antd` 将会默认采用 `light` 主题，

这对于一些开启 `dark` 主题的用户很不友好。

因此我们需要创建一个公用的 `ConfigProvider` 组件，用于解决后续编写 vue 组件时遇到的 theme 问题。

创建 `docs/vue/ConfigProvider.vue` 内容如下

```vue
<template>
  <ConfigProvider
    :theme="{
      algorithm: isDark ? darkAlgorithm : defaultAlgorithm,
      token: {
        colorPrimary: colorPrimary,
        colorLink: colorPrimary,
      },
    }"
  >
    <slot />
  </ConfigProvider>
</template>

<script lang="ts" setup>
import { colorPrimary } from "./const";
import { ConfigProvider, theme } from "ant-design-vue";
import { useData } from "vitepress";

const { darkAlgorithm, defaultAlgorithm } = theme;

const { isDark } = useData();
</script>
```

以后编写的 vue 组件只需要放到 `<ConfigProvider>` 中即可自动根据 `vitepress` 切换主题。

## 3. 创建分享链接组件

新建 `docs/vue/ShareLink.vue` 内容如下

```vue
<template>
  <ConfigProvider>
    <Paragraph :copyable="{ text: shareLink }">复制分享链接</Paragraph>
  </ConfigProvider>
</template>

<script lang="ts" setup>
import { ConfigProvider } from "./ConfigProvider.vue";
import { Typography } from "ant-design-vue";
import { ref, onMounted } from "vue";
const { Paragraph } = Typography;

const shareLink = ref("");

onMounted(() => {
  shareLink.value = window.location.href;
});
</script>
```

## 4. 创建 shareLink 模版文件

虽说可以直接在每个页面的 markdown 中通过 vue 引入 `ShareLink.vue` 组件，但是每个页面都这样写还是有些麻烦。我们将它写成一个公用的 markdown 模版，其它页面只需要引入即可。

新建 `/docs/templates/ShareLink.md` 内容如下

```markdown
<ShareLink/>

::: info
分享给你的好友
:::

<script setup>
import ShareLink from '../vue/ShareLink.vue';
</script>
```

## 5. 在页面中引用

现在如果需要在页面底部显示一个分享链接，我们只需要这样做

```markdown
todo 编写你的文档

<!--@include: ../templates/ShareLink.md-->
```

# 图片点击放大功能

`vitepress` 不像 `vuepress` 内置图片放大功能，如果没有这个功能，在文章内查看图片特别别扭。这里使用 `medium-zoom` 来处理图片放大。

## 1. 安装依赖

```shell
pnpm add medium-zoom -D
```

## 2. 配置 vitepress 主题

修改 `docs/.vitepress/theme/index.ts` （如果没有就新建它）

内容如下

```typescript
import mediumZoom from "medium-zoom";
import "medium-zoom/dist/style.css";
import type { Theme } from "vitepress";
import { useRoute } from "vitepress";
import DefaultTheme from "vitepress/theme";
import { h } from "vue";
import { onMounted, watch, nextTick } from "vue";

export default {
  extends: DefaultTheme,
  Layout: () => {
    return h(DefaultTheme.Layout, null, {
      // https://vitepress.dev/guide/extending-default-theme#layout-slots
    });
  },
  enhanceApp({ app, router, siteData }) {
    // ...
  },
  setup() {
    //  添加以下代码 --》
    const route = useRoute();
    const initZoom = () => {
      mediumZoom(".content-container p img", {
        background: "var(--vp-c-bg)",
        container: document.body,
      });
    };
    onMounted(() => {
      initZoom();
    });
    watch(
      () => route.path,
      () => nextTick(() => initZoom())
    );
    //  《--- 结束
  },
} satisfies Theme;
```

## 3. 修复 medium-zoom 放大图片导致的图片被遮挡问题

解决这个问题只需要加个全局样式即可

```css
.medium-zoom-image--opened {
  z-index: 999999;
}
```
