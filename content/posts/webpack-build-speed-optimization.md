---
title: "自行车变摩托: 如何将老旧Webpack项目的编译时间从14秒提速至闪电般的80毫秒！"
date: 2023-11-30T17:59:46-05:00
lastmod: 2023-11-30T17:59:46-05:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "突破性能瓶颈：一文带你了解如何将Webpack构建速度提升至原来的160倍！深入探索实用技巧和精细调整，让你的项目编译时间从14秒降至令人难以置信的80毫秒。立即学习，让项目构建效率飞跃，告别等待的烦恼。（ChatGPT 给的描述）"
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

最近接手了一个别人开发的老 webpack 项目，在体验过其堪比龟速的启动、编译和打包之后，决定对 webpack 进行优化。堪比龟速的 webpack 配置，浪费的可是寸金难买的时间啊！

# 速度对比

> 以下测试是在 Macbook pro M1 (2020) 16GB 上进行测试，通过 3 次测试取的平均值

|        | 启动速度 | 编译速度 | 打包速度 |
| ------ | -------- | -------- | -------- |
| 优化前 | 15s      | 14s      | 26s      |
| 优化后 | 2s       | 80ms     | 5s       |

# 问题分析

要先解决速度问题，首先得知道 webpack 编译过程中，到底慢在了哪一个环节。我使用了以下两个 webpack 插件进行分析.

## [speed-measure-webpack-plugin](https://www.npmjs.com/package/speed-measure-webpack-plugin)

这个插件用于显示 webpack 在编译过程中每个步骤所执行的时间。（具体如何使用请执行查看官方文档）

下面是项目启动时的耗时。

```text
 SMP  ⏱
General output time took 14.25 secs

 SMP  ⏱  Plugins
CopyPlugin took 0.015 secs

 SMP  ⏱  Loaders
ts-loader took 8.35 secs
  module count = 87
babel-loader took 4.95 secs
  module count = 55
modules with no loaders took 4.58 secs
  module count = 3289
css-loader took 0.346 secs
  module count = 18
style-loader, and
css-loader took 0.008 secs
  module count = 18

assets by path js/*.js 41.7 MiB 13 assets
assets by path music/*.mp3 1.17 MiB 12 assets
assets by path images/*.png 172 KiB
```

分析上面的记录，发现有 3 点可疑的地方

1. ts-loader 8.35s 这完全不合理，项目中就最多几十个文件，不应该需要这么久
2. babel-loader 4.95s 这个时间还是有点长，几十个文件就算全部经过 babel 转换，也不应该这么久
3. js/\*.js 文件达到了 41.7 MB, 其中一个 js 文件达到了 28.1MB （这比我这辈子写的代码量都多）

## [webpack-bundle-analyzer](https://www.npmjs.com/package/webpack-bundle-analyzer)

顺着 28.1MB 这个线索，因为这个项目压缩后的逻辑代码顶多也就 500kb，所以我推断出肯定在打包时打包进去了一些没用的模块。通过 `webpack-bundle-analyzer` 插件，来分析下到底有哪些模块被打包了进去。（具体使用请查看官方文档）

下面是打包后的模块关系图。
![img](https://i.imgur.com/fAZatRz.png)

从图中可以分析出到 antd 和 antd icons 这俩包是没经过 tree shaking 直接全部打包进了逻辑代码中。而且其它的一些外部包，也被打包了进去，这是我不想看到的。每次编译都要重新编译一遍这些外部库（虽然 webpack 提供 cache 功能但是也不太理想），大部分时间应该是浪费在了这些外部库的编译上。理想情况下外部库应该和逻辑代码拆分出来，这样在编译时，只编译逻辑代码，这样应该会提速不少。

# 解决问题

分析出了导致龟速的原因，现在来具体解决这些问题。

## 更新依赖

由于项目比较老旧，更新 webpack 和一些其它库的版本。这个很有必要，因为新版本中一般都会有性能优化。

这里使用 [npm-check](https://www.npmjs.com/package/npm-check) 来检查哪些包有新版本可用。

安装

```shell
pnpm add npm-check
```

在 `package.json` 中编写一个检测命令

```json
{
  "scripts": {
    "checkUpdate": "NPM_CHECK_INSTALLER=pnpm npm-check -u -i"
  }
}
```

> 由于该项目使用 pnpm 所以在命令中添加了 `NPM_CHECK_INSTALLER=pnpm` 用于指定安装器，如果你用 `yarn` 可以修改为 `NPM_CHECK_INSTALLER=yarn`

检查依赖更新

```shell
pnpm run checkUpdate
```

执行后，它将会检查所有依赖中是否有可用的新版本，只需要按下 `Space` 选择你要更新的依赖，然后回车进行更新即可。（大版本更新可能会存在不兼容问题，这个需要自己解决）

## 解决 ts-loader 运行过慢问题

来看看 `webpack` 的配置文件中关于 `ts-loader` 的配置

```javascript
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        exclude: /node_modules/,
        loader: "ts-loader",
      },
    ],
  },
  //  ...
};
```

还有 `tsconfig.json` 中的配置

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "commonjs",
    "checkJs": false,
    "jsx": "react",
    "allowJs": true,
    "esModuleInterop": true,
    "sourceMap": false
  },
  "include": ["src"]
}
```

这俩文件的配置都很简洁，可以说完全使用的是 `ts-loader` 的默认配置，但是这也是恰恰导致 `ts-loader` 执行缓慢的问题。

首先如果按照这样的配置，在 `ts-loader` 执行时，会调用 `typescript` 进行编译与类型检查，这不是不行，但是你可以想象每次编译时都要把几十个文件重新通过 `typescript` 编译，然后再进行类型检查，这不是浪费时间吗？我们使用 `typescript` 主要是为了保证类型安全和优化类型提示，现在都快 2024 年了，这些工作应该交给代码编辑器去做，而不是通过 webpack 做。

我们期望的应该是我写完代码，还未保存时编辑器就能提示我是否有错误，而不是在 `webpack` 通过 `ts-loader` 编译后再提示我有没有错误。因此以上的配置不仅抢了代码编辑器的工作，而且效率还低。

因此我们修改下配置

```javascript
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.tsx?$/,
        exclude: /node_modules/,
        loader: "ts-loader",
        options: {
          //  告诉 typescript，不要执行其常规的类型检查只进行转译
          transpileOnly: true,
        },
      },
    ],
  },
  //  ...
};
```

修改完配置后，我们再使用 `webpack` 进行编译，此时 `ts-loader` 的耗时已经降到了 `0.13s`. 就一个 `transpileOnly` 配置，让它快了 `64` 倍.

## 解决 babel-loader 运行过慢问题

首先看看 webpack 中的配置

```javascript
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        exclude: /node_modules/,
        use: {
          loader: "babel-loader",
        },
      },
    ],
  },
  //  ...
};
```

和下面`.babelrc` 的配置

```json
{
  "presets": ["@babel/preset-env", "@babel/preset-react"],
  "plugins": [["@babel/plugin-transform-runtime"]]
}
```

又是官方提供的 Demo 中的默认配置，但是这也是导致它执行缓慢的问题，这里使用的是 `@babel/preset-env` 它默认会将代码转换成 `es5`，即使我们的程序需要兼容低版本浏览器，它也不应该这样配置（@babel/preset-env 应该只运作与 production 而不是 development，因为我们都是用最新浏览器进行开发的，所以此时的 @babel/preset-env 就是鸡肋）。

这样配置导致几十个文件，每次编译都要经过 `babel-loader` 将 `es6` 代码转换成 `es5`，这样对于我们日常开发来说效率很低。

由于这个项目是给内部使用，所以不用考虑低版本浏览器问题，因此这里我们直接去掉 `@babel/preset-env` 只让 `babel-loader` 做 `jsx` 文件的转换。

```javascript
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        exclude: /node_modules/,
        use: {
          loader: "babel-loader",
          options: {
            presets: ["@babel/preset-react"],
          },
        },
      },
    ],
  },
  //  ...
};
```

> 我们在 `webpack.config.js` 中配置了 `babel-loader` 的 options，
> 因此就可以删掉 `.babelrc` 文件了

修改完配置后，我们再使用 `webpack` 进行编译，此时 `babel-loader` 的耗时已经降到了 `0.54s`. 就一个 `transpileOnly` 配置，让它快了 `9` 倍.

## 解决 externals 问题

> webpack externals 配置如何使用，请到官网查看

正如上文提到的，一些外部依赖包在每次编译时都会被编译到主逻辑代码中。这样是不合理的，我们期望的是只编译逻辑代码，因为外部依赖是不需要改变，也不需要重新编译的。

一般情况下我们使用 webpack 的 `externals` 配置将这些外部依赖包全部剔除出去。并且在线上版本引用 `cdn` 上的外部依赖包。但是由于这个项目是一个浏览器扩展，因此它无法引用远程文件（CWS 政策），因此我们希望将这些 `externals` 打包到 `externals.js` 文件中，并且将所有的依赖全部注入到 `window` 变量中。这样我们只需要在主逻辑代码运行之前先引入这个 `externals.js` 文件即可。

### 创建 externals 配置文件

```javascript
//  src/externalsConfig.js
const { name: pkgName } = require("../package.json");

const externals = [
  "antd",
  "@ant-design/icons",
  "@material-ui/icons",
  "@material-ui/core",
  "@material-ui/lab",
  "react",
  "react-dom/client",
  "react/jsx-runtime",
  "@react-hook/resize-observer",
  "emoji-picker-react",
  "jspreadsheet-ce",
  "react-beautiful-dnd",
  "react-rnd",
  "react-virtualized-auto-sizer",
  "react-window",
  "react-dom",
];

//  由于依赖包中存在特殊字符，如果直接注入到 window 上可能会出错
//  同时为了避免包冲突问题，这里加上了当前项目的 name 并且将依赖包中的特殊字符住转换成下划线
function getExternalInjectName(name) {
  return pkgName.replace(/[-\/@]/g, "_") + "_" + name.replace(/[-\/@]/g, "_");
}

module.exports = {
  externals,
  //  返回 webpack.externals 所需的配置
  getWebpackExternalsConfig() {
    return externals.reduce((prev, current) => {
      prev[current] = `global ${getExternalInjectName(current)}`;
      return prev;
    }, {});
  },
  getExternalInjectName,
};
```

### 配置 webpack.config.js

```javascript
const { getWebpackExternalsConfig } = require("./src/externalsConfig.js");
module.exports = {
  //  ...
  externals: {
    ...getWebpackExternalsConfig(),
  },
};
```

在配置完后，我们再次进行打包，之前 28.1MB 的文件，现在已经只有 508KB 了，也就说这些依赖包就占用快 28MB。

### 将 externals 打包成一个独立的文件

首先我们写一个脚本文件，用于将 externals 打包成独立文件

```typescript
//  scripts/buildExternals.ts
import { externals, getExternalInjectName } from "../src/externalsConfig";
import { writeFileSync, unlinkSync } from "fs";
import { resolve } from "path";
import webpack from "webpack";

const entryPath = resolve(__dirname, ".", ".external.tmp.js");

//  写入临时的 externals.js 用于给 webpack 进行打包
{
  let externalContent = ``;
  for (let moduleName of externals) {
    externalContent += `const ${getExternalInjectName(
      moduleName
    )} = require("${moduleName}");\n`;
  }

  for (let moduleName of externals) {
    externalContent += `window["${getExternalInjectName(
      moduleName
    )}"] = ${getExternalInjectName(moduleName)};\n`;
  }

  writeFileSync(entryPath, externalContent, "utf-8");
}

//  打包 webpack
{
  console.log("开始打包 externals");
  const compiler = webpack({
    entry: entryPath,
    output: {
      filename: "externals.js",
      path: resolve(__dirname, "..", "dist"),
      iife: true,
    },
    mode: "production",
  });

  compiler.run((err, stats: any) => {
    if (err) {
      console.error(err);
      return;
    }

    console.log("externals 打包完成");

    console.log(
      stats.toString({
        chunks: false, // Makes the build much quieter
        colors: true, // Shows colors in the console
      })
    );

    //  删除临时文件
    unlinkSync(entryPath);
  });
}
```

然后我们在 `package.json` 中编写一个用于打包 externals 的命令

```json
{
  "scripts": {
    "build:externals": "esno ./scripts/buildExternals.ts"
  }
}
```

现在我们只需要执行下

```shell
pnpm run build:externals
```

就可以在将所有的 externals 打包到 `dist/externals.js` 中。这样在主逻辑执行前，将这个文件提前引入即可。

通过以上三个步骤的优化，我们成功把编译时间从 `14s` 变成了 `80ms`!
