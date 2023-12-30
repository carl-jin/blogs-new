---
title: "I18next 在 React 如何使用 Typescript 编写？"
date: 2023-12-29T14:43:08-05:00
lastmod: 2023-12-29T14:43:08-05:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "对于没有用过 i18n 库的开发者来说，刚开始接触还是有点蒙，本文将通过一个简单的例子演示如何在 react 中使用 i18next 并且提供 typesafe"
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

> react^18.2.0
> 
> i18next^23.7.12
> 
> react-i18next^14.0.0

# 安装依赖

```shell
pnpm add i18next react-i18next -D
```

> i18next 是国际化解决方案，它支持所有的 js 框架
> react-i18next 是为了在 react 中使用 hook 或者 HOC 的方式来调用 i18next

# 创建语言包

来创建中文和英文的语言包

```text
📦locales
 ┣ 📂en-US
 ┃ ┣ 📜common.json
 ┃ ┗ 📜user.json
 ┗ 📂zh-CN
 ┃ ┣ 📜common.json
 ┃ ┗ 📜user.json
```

> 目录结构和文件名称可以按照你的业务需求改，这里只是为了做演示

上面这个目录结构中，`zh-CN` 和 `en-US` 俩目录代表着支持的语言代码，而 `common.json` 和 `user.json` 是针对翻译的命名空间分割。当项目复杂时就有命名空间的概念，比如一些常用的翻译都放在 `common.json` 而针对用户的翻译都放在 `user.json` 中，这样方便管理

来看下 `zh-CN/user.json` 的内容

```json
{
  "nameFieldName": "名称:",
  "nameFieldPlaceholder": "请输入您的中文名称"
}
```

对应的 `en-US/user.json` 的内容

```json
{
  "nameFieldName": "Name:",
  "nameFieldPlaceholder": "Pls Enter Your English name"
}
```

# 初始化 i18next

创建个 `i18next.ts`

```typescript
import i18n from "i18next";
import { initReactI18next } from "react-i18next";
import zhCNCommon from "./locales/zh-CN/common.json";
import zhCNUser from "./locales/zh-CN/user.json";
import enUSCommon from "./locales/en-US/common.json";
import enUSUser from "./locales/en-US/user.json";

i18n
  // 初始化 react-i18next
  .use(initReactI18next)
  .init({
    //  支持的命名空间
    ns: ["common", "user"],
    //  翻译资源
    resources: {
      "zh-CN": {
        common: zhCNCommon,
        user: zhCNUser,
      },
      "en-US": {
        common: enUSCommon,
        user: enUSUser,
      },
    },
    //  默认语言
    lng: "zh-CN",
  });
```

# 使用

将 `i18next.ts` 引入到你的项目入口文件中，然后就可以在 react 组件中使用 i18next 了

```tsx
import { useTranslation } from "react-i18next";

function NameInput() {
  //  在调用 useTranslation 时需要传入命名空间
  const { t } = useTranslation("common");
  return (
    <>
      {t("nameFieldName")}
      <input type="text" placeholder={t("nameFieldPlaceholder")} />
    </>
  );
}
```

如果你想切换语言可以这样

```typescript
const { i18n } = useTranslation("common");
i18n.changeLanguage("en-US");
```

# 进阶内容

## 1. 优化 i18next 初始化文件中 resources 的定义

虽然我们现在将翻译放到 json 中，但是现在这种引入方式很不舒服，每次新增新的语言翻译命名空间都需要写一个 import，如果提供几十个语言的翻译，这将是很头痛的问题

```typescript
import zhCNCommon from "./locales/zh-CN/common.json";
import zhCNUser from "./locales/zh-CN/user.json";
import enUSCommon from "./locales/en-US/common.json";
import enUSUser from "./locales/en-US/user.json";
```

理想情况下应该能自动从 `locales` 中读取每个语言和每个语言下的命名空间，自动的生成 `resources` 配置

现在写个脚本 `loadLangsToResouerce.ts`, 来读取所有的语言包 json 并且将它们转换成对应的 `resources` 数据

> 下面的 import.meta.glob 是 vite 提供的方法

```typescript
//  加载所有语言包
const modules = import.meta.glob("./locales/**/*.json", {
  eager: true,
}) as Record<string, { default: never }>;

export const localeTransitions = Object.entries(modules).reduce(
  (prev, current) => {
    const [path, module] = current;
    const lang = path.match(/\/locales\/([\w-]+)\//);
    const filename = path.match(/\/([\w-_]+)\.json$/);

    if (filename && lang) {
      prev[lang[1]] = prev[lang[1]] || {};
      prev[lang[1]][filename[1]] = module.default;
    } else {
      console.error(`无法解析文件名称 path:${path}`);
    }

    return prev;
  },
  {}
);
```

然后我们重新改写下 `i18next.ts`

```typescript
import i18n from "i18next";
import { initReactI18next } from "react-i18next";
import { localeTransitions } from "./loadLangsToResouerce.ts";

i18n
  // 初始化 react-i18next
  .use(initReactI18next)
  .init({
    ns: Object.keys(localeTransitions),
    resources: localeTransitions,
    //  默认语言
    lng: "zh-CN",
  });
```

这样就简洁多了

## 2. i18next 多实例问题

目前的 i18next 的初始化是针对全局的，如果一个项目中存在多个 i18next 时就会报错。
这种情况应该尽量避免，因为我们自己开发的组件库中可能会存在这个问题。让我们来改写下 `i18next.ts` 让它孤立起来

```typescript
import i18n from "i18next";
import { initReactI18next } from "react-i18next";
import { localeTransitions } from "./loadLangsToResouerce.ts";

const i18nInstance = i18n.createInstance();

i18nInstance
  // 这里就不能初始化 reactI18next 了
  // .use(initReactI18next)
  .init({
    ns: Object.keys(localeTransitions),
    resources: localeTransitions,
    //  默认语言
    lng: "zh-CN",
  });

export default i18nInstance;
```

现在我们在需要使用这个 i18next 的组件外围包裹一个 `I18nextProvider`，这样代替了之前的`.use(initReactI18next)`使得组件内部依然可以 h 正确使用 `useTranslation` 这个 hook

```tsx
import i18nInstance from "./i18next";
import UserForm from "./UserForm";

export function UserPage() {
  return (
    <I18nextProvider i18n={i18nInstance} defaultNS={"common"}>
      <UserForm />
    </I18nextProvider>
  );
}
```

现在就可以在 `UserForm` 使用了

```tsx
function UserForm() {
  //  这里我们就不需要传入 'common' 了
  //  在 I18nextProvider 中已经指定了默认的命名空间
  // const { t } = useTranslation('common');

  //  如果你需要使用 'user' 的命名空间也可以这样
  // const { t } = useTranslation('user');

  const { t } = useTranslation();
  return (
    <>
      {t("nameFieldName")}
      <input type="text" placeholder={t("nameFieldPlaceholder")} />
    </>
  );
}
```

## 3. typescript 支持

现在调用 `t("nameFieldName")` 还没有任何的提示，[按照官方的文档](https://www.i18next.com/overview/typescript) 来写一个 `i18next.d.ts` 文件来声明类型

```typescript
import "i18next";
import { localeTransitions } from "./loadLangsToResouerce.ts";

declare module "i18next" {
  interface CustomTypeOptions {
    defaultNS: "zh-CN";
    resources: (typeof localeTransitions)["zh-CN"];
  }
}
```

# 扩展内容

如果你想实现下面这样的提示效果
![https://i.imgur.com/CPk33SB.png](https://i.imgur.com/CPk33SB.png)

可以试试 [vite-i18n-gen-resources-type](https://github.com/CRMChatReactComponent/vite-i18n-gen-resources-type) 这个 vite 插件
