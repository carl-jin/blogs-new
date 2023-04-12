---
title: "Electron 自动更新实现"
date: 2023-04-11T21:23:18-04:00
lastmod: 2023-04-11T21:23:18-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "如何在 Electron 实现自动更新，并且支持所有系统的更新"
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

# 提示

本文适用于内容使用的软件，因为需要禁用 `asar` 和关闭代码签名，因此只适用于内部使用软件，如果你的软件是需要上架或者要加密代码，那么本文并`不适合`你。

# 背景

现在需要开发一个内部软件，不需要上架，也不需要加密代码，为了快捷开发，我们选择了 Electron。因为软件频繁更新，如果每次更新都打包成对应的三个平台再去分发下载，每次都几百兆，这太不理想了。

网上现有的解决方案大多数都是针对需要上架到平台的自动更新解决方案，都不是太理想。我们理想的方法，就是尽量的减少更新文件的大小，最好能像网页一样刷新一下就能得到最新版本。

让我们看看如何实现

# 项目目录

我使用了 [https://github.com/cawa-93/vite-electron-builder](https://github.com/cawa-93/vite-electron-builder) 作为模版，它打包出来的文件大概是下面这样。

```text
- app electron 执行目录
-- node_modules  主进程的依赖模块
-- packages 项目的逻辑代码
--- main    主进程的逻辑代码
--- preload  preload
--- renderer  渲染进程代码
-- package.json
```

`package.json` 中的内容如下

```json
{
  "main": "./packages/main/dist/index.cjs"
}
```

要运行内容只需要执行 `electron .` 即可

# 关闭 asar

首先我们需要关闭 asar，因为我们希望更新的代码能直接覆盖掉之前的老代码，关闭 asar 是最快的解决方法，虽然使用 asar unpack 也可以达到，但是还是相对于麻烦。

`asar: false` 我们在配置里面，将 asar 关闭即可

# 写个脚本检查更新

```typescript
import { app } from "electron";
import { get } from "node:https";
import { join } from "node:path";
import {
  readFileSync,
  existsSync,
  mkdirSync,
  readdirSync,
  statSync,
} from "node:fs";
import { isEqual } from "lodash";
import unzipper from "unzipper";
import { fork } from "node:child_process";

const REMOTE_URI = "https://你的服务器链接，建议使用 netlify";

export function autoUpdate() {
  if (!import.meta.env.PROD) return;
  setInterval(() => {
    checkUpdate();
    //  每六个小时检查一次
  }, 1000 * 60 * 60 * 6);

  //  程序运行 2 秒后，检查更新
  setTimeout(() => {
    checkUpdate();
  }, 1000 * 2);
}

export async function installNewVersion() {
  if (!import.meta.env.PROD) return;
  const appPath = app.getAppPath();
  fork(join(appPath, "autoUpdateInstaller", "installer.cjs"), {
    cwd: appPath,
    stdio: "inherit",
  });

  setTimeout(() => {
    app.quit();
    process.exit(0);
  }, 5000);
}

export async function checkIfHaveNewUpdate(): Promise<boolean> {
  if (!import.meta.env.PROD) return false;
  const currentVersion = app.getVersion();
  const pkg: any = await fetchRemotePkgJSON();
  if (!pkg) return false;

  const isHaveNewVersion =
    pkg.version !== currentVersion &&
    checkIfNeedUpdate(currentVersion, pkg.version);
  isHaveNewVersion && checkUpdate();

  return isHaveNewVersion;
}

export async function checkUpdate() {
  if (!import.meta.env.PROD) return;
  const currentVersion = app.getVersion();
  //  去服务器上获取新版本

  try {
    const pkg: any = await fetchRemotePkgJSON();
    if (!pkg) return;

    if (
      pkg.version !== currentVersion &&
      checkIfNeedUpdate(currentVersion, pkg.version)
    ) {
      const appPath = app.getAppPath();
      const currentPkgJSON = JSON.parse(
        readFileSync(join(appPath, "package.json"), "utf-8")
      );
      const unzipPath = join(appPath, "unzip");

      // 如果路径存在，清空它
      if (existsSync(unzipPath)) {
        rmdir(unzipPath);
      }
      mkdirSync(unzipPath);

      //  判断依赖是否一样，如果一样就下载 app.zip
      //  否则就下载包含 node_modules 的 fullSizeApp.zip 包
      let downloadZipPath = `${REMOTE_URI}/app.zip`;
      if (
        !isEqual(
          Object.keys(pkg.dependencies),
          Object.keys(currentPkgJSON.dependencies)
        )
      ) {
        downloadZipPath = `${REMOTE_URI}/fullSizeApp.zip`;
      }

      get(downloadZipPath, (response) => {
        response
          .pipe(unzipper.Extract({ path: unzipPath }))
          .on("finish", () => {
            console.log("解压缩完成！");
            //  todo 通知前端，有新版本
          })
          .on("error", (err: any) => {
            console.error("解压缩过程中出现错误：" + err.toString());
          });
      }).on("error", (err) => {
        console.error("下载 ZIP 文件出错：" + err.toString());
      });
    }

    if (!pkg) return;
  } catch (e) {
    console.error(e);
  }
}

function fetchRemotePkgJSON(): Promise<Object> {
  return new Promise<Object>((res, rej) => {
    get(`${REMOTE_URI}/package.json`, (_res) => {
      let data = "";

      // 处理响应数据
      _res.on("data", (chunk) => {
        data += chunk;
      });

      // 响应数据接收完毕
      _res.on("end", () => {
        try {
          res(JSON.parse(data));
        } catch (e: any) {
          systemLogger.error(
            `自动更新检查请求失败 远程 PKG 解析错误:` + e.toString()
          );
          //  @ts-ignore
          res(null);
        }
      });
    }).on("error", (err) => {
      rej(`自动更新检查请求失败:` + err.toString());
    });
  });
}

//计算版本号大小,转化大小
function toNum(a: string) {
  let c = a.split(".");
  let num_place = ["", "0", "00", "000", "0000"],
    r = num_place.reverse();
  for (let i = 0; i < c.length; i++) {
    let len = c[i].length;
    c[i] = r[len] + c[i];
  }
  let res = c.join("");
  return res;
}

function checkIfNeedUpdate(localVersion: string, remoteVersion: string) {
  let a = toNum(localVersion);
  let b = toNum(remoteVersion);
  if (a == b) {
    //  版本号相同
    return false;
  } else if (a > b) {
    return false;
  } else {
    return true;
  }
}

function rmdir(dir: string) {
  var list = readdirSync(dir);
  for (let i = 0; i < list.length; i++) {
    let filename = join(dir, list[i]);
    let stat = statSync(filename);

    if (filename == "." || filename == "..") {
      // pass these files
    } else if (stat.isDirectory()) {
      // rmdir recursively
      rmdir(filename);
    } else {
      // rm fiilename
      unlinkSync(filename);
    }
  }
  rmdirSync(dir);
}
```

让我们来梳理下逻辑，首先程序开始运行时候，我们只需要调用下 `autoUpdate` 方法即可，该方法会调用 `checkUpdate` 去远程服务器判断是否有新版本的更新。

我们通过检查远程服务器上的 `package.json` (过后会讲到如何生成) 与当前系统运行的 package.json 的版本是否一致，如果不一致则去服务器拉去新的代码，并且解压到一个叫

`unzip` 的文件夹下面，然后通知前端进行更新。

当前端选择进行更新时，我们只需要将 unzip 中的文件，替换掉整个 app 目录即可。

# 代码替换

首先我们来考虑一个问题

如果每次更新代码，都要重新更新下 node_modules 这很明显不合适，node_modules 里面的文件基本上不会更新，而且还很大；

但是有些情况还必须要更新 node_modules，比如引用了新的包之类的。

因此判断本地运行软件中的 package.json 是否与远程的 package.json 一致，如果一致就不更新 node_modules， 否则就全部更新

```typescript
let downloadZipPath = `${REMOTE_URI}/app.zip`;
//  如果不相等，下载包含 node_modules 的 zip 文件
if (
  !isEqual(
    Object.keys(pkg.dependencies),
    Object.keys(currentPkgJSON.dependencies)
  )
) {
  downloadZipPath = `${REMOTE_URI}/fullSizeApp.zip`;
}
```

代码替换通过一下方法进行执行

这里我们通过 `autoUpdateInstaller/installer.cjs` 来进行代码替换，

我们没写到 main 的原因是因为代码替换时，如果替换本身可能会出现文件被占用问题，因此最好独立出来

```typescript
export async function installNewVersion() {
  if (!import.meta.env.PROD) return;
  const appPath = app.getAppPath();
  //  注意这里需要用 fork
  //  如果直接执行 spawn node ./autoUpdateInstaller/installer.cjs
  //  会导致打包后，node 执行环境找不到问题
  fork(join(appPath, "autoUpdateInstaller", "installer.cjs"), {
    cwd: appPath,
    stdio: "inherit",
  });

  setTimeout(() => {
    app.quit();
    process.exit(0);
  }, 5000);
}
```

我们来看看 `installer.cjs` 里面的内容

```typescript
/**
 * 处理自动更新安装
 */

const fs = require("fs");
const path = require("path");

//  等待下 electron 关闭
setTimeout(() => {
  try {
    const unzipFolder = path.resolve(__dirname, "..", "unzip");
    const targetFolder = path.resolve(__dirname, "..");
    if (fs.existsSync(unzipFolder)) {
      //  代码替换
      copyFolderRecursiveSync(unzipFolder, targetFolder);
    }
  } catch (e) {
    fs.writeFileSync(
      path.resolve(__dirname, "error.txt"),
      e.toString(),
      "utf-8"
    );
  }
}, 1);

// 复制文件夹及其内容的函数
function copyFolderRecursiveSync(source, target) {
  // 如果目标目录不存在，则创建目标目录
  if (!fs.existsSync(target)) {
    fs.mkdirSync(target);
  }

  // 获取源目录的文件列表
  const files = fs.readdirSync(source);

  // 遍历文件列表，处理每个文件或子目录
  files.forEach((file) => {
    const sourcePath = path.join(source, file);
    const targetPath = path.join(target, file);

    // 如果当前文件是文件夹，则递归复制文件夹
    if (fs.statSync(sourcePath).isDirectory()) {
      copyFolderRecursiveSync(sourcePath, targetPath);
    } else {
      // 否则，复制文件
      fs.copyFileSync(sourcePath, targetPath);
    }
  });
}
```

现在我们已经完成了检查更新和代码替换的功能。

但是如何生成 app.zip 或者 fullSizeApp.zip 文件呢？让我们接着往下看

# 发布版本

由于我们的目录结构如下，所有的逻辑代码都放在 packages 文件夹里面，因此每次我们只需要将 app 文件夹打包，然后发布就行了

```text
- app electron 执行目录
-- node_modules  主进程的依赖模块
-- packages 项目的逻辑代码
--- main    主进程的逻辑代码
--- preload  preload
--- renderer  渲染进程代码
-- package.json
```

但是我们刚刚提到，node_modules 应该是只有依赖包发生改变才进行更新，而不是每次都进行更新。但是考虑到某些情况下还是有 node_modules 更新的情况。

因此我们需要打包成两个文件，一个是 app.zip 只包含 package.json 和 packages 文件夹，

另外一个是 fullSizeApp.zip ，它包含了整个 app 文件夹里面的内容，

这样只需要在 main 对照下远程的版本依赖和本地的版本依赖是否相同，就可以直接判断是否要下载 node_modules 了。

那么问题来了，我们该如何得到 app 目录呢？

它就在每次你编译完 electron 后的软件文件夹里面，比如我用的是 M1 的芯片，编译完后的位置就在下面

```text
dist/mac-arm64/软件名称/Contents/Resources/app
```

因此我们在发布更新包之前，必须要先进行一次编译才能获取到这个文件夹，

这个也很简单，我们来写个命令就行了, 其中

`pnpm run build` 是打包逻辑文件，比如 main 和 renderer，

`pnpm run compile:clear` 是清理 dist 目录中之前编译的包

`pnpm run compile:mac` 这是进行 mac 下的编译

```json
{
  "beforeStartAutoUpdate": "pnpm run build && pnpm run compile:clear && pnpm run compile:mac"
}
```

现在我们能通过 `pnpm run beforeStartAutoUpdate` 来得到 app 目录了，

让我们看看如何将它压缩成 `app.zip` 和 `fullSizeApp.zip`

我们来写个脚本

```typescript
import inquirer from "inquirer";
import {
  readFileSync,
  existsSync,
  createWriteStream,
  readdirSync,
  statSync,
  unlinkSync,
  rmdirSync,
  copyFileSync,
} from "node:fs";
import { join, basename, resolve } from "node:path";
import { fileURLToPath } from "node:url";
import { spawn } from "node:child_process";
import archiver from "archiver";
import { mkdirSync } from "fs";

const __dirname = fileURLToPath(new URL(".", import.meta.url));

(async () => {
  const pkgJSON = JSON.parse(
    readFileSync(resolve(__dirname, "../package.json"), "utf-8")
  );

  const answers = await inquirer.prompt([
    {
      type: "confirm",
      name: "continue",
      message: `当前的 package.json 版本是 ${pkgJSON.version}，是否继续执行？`,
      default: false,
    },
  ]);

  if (!answers.continue) return;

  //  判断下需要自动部署的文件是否存在，如果不存在，需要让它自己先执行下 compile:mac
  const appFolder = resolve(
    __dirname,
    "..",
    "dist",
    "mac-arm64",
    "软件名称",
    "Contents",
    "Resources",
    "app"
  );

  if (!existsSync(appFolder))
    throw new Error(
      `文件夹不存在，请先尝试执行 pnpm run beforeStartAutoUpdate, ${appFolder}`
    );

  const zipFolder = resolve(__dirname, "..", "dist", "autoUpdate");
  const outputFullSizeZipPath = resolve(zipFolder, "fullSizeApp.zip");
  const outputZipPath = resolve(zipFolder, "app.zip");

  //  判断 compile 的 package json 和 当前开发环境的是否一样
  const compilePkgJSON = JSON.parse(
    readFileSync(join(appFolder, "package.json"), "utf-8")
  );
  if (pkgJSON.version !== compilePkgJSON.version) {
    throw new Error(
      `编译后的 package.json 和当前不一致，请检查是否是新版本，当前${pkgJSON.version}, compiled ${compilePkgJSON.version}`
    );
  }

  if (existsSync(zipFolder)) {
    rmdir(zipFolder);
  }
  mkdirSync(zipFolder);

  await archive(outputFullSizeZipPath, [
    join(appFolder, "node_modules"),
    join(appFolder, "packages"),
    join(appFolder, "package.json"),
  ]);
  await archive(outputZipPath, [
    join(appFolder, "packages"),
    join(appFolder, "package.json"),
  ]);

  //  将 package.json 放到 outputZipPath 里面
  copyFileSync(
    join(appFolder, "package.json"),
    join(zipFolder, "package.json")
  );

  function archive(zipPath: string, files: string[]) {
    return new Promise<void>((res, rej) => {
      // 创建一个可写流来写入 zip 文件
      const output = createWriteStream(zipPath);
      const archive = archiver("zip", {
        zlib: { level: 9 },
      });

      // 当打包完成时触发 'close' 事件
      output.on("close", function () {
        console.log(archive.pointer() + " total bytes");
        console.log(
          "archiver has been finalized and the output file descriptor has closed."
        );
        res();
      });

      // 当出现错误时触发 'error' 事件
      archive.on("error", function (err: any) {
        rej(err);
      });

      // 完成打包并关闭输出流
      archive.pipe(output);

      // 将指定文件夹添加到 zip 文件中
      for (let filePath of files) {
        if (~filePath.indexOf("package.json")) {
          archive.file(filePath, { name: basename(filePath) });
        } else {
          archive.directory(filePath, basename(filePath));
        }
      }

      archive.finalize();
    });
  }
})();

function rmdir(dir: string) {
  var list = readdirSync(dir);
  for (let i = 0; i < list.length; i++) {
    let filename = join(dir, list[i]);
    let stat = statSync(filename);

    if (filename == "." || filename == "..") {
      // pass these files
    } else if (stat.isDirectory()) {
      // rmdir recursively
      rmdir(filename);
    } else {
      // rm fiilename
      unlinkSync(filename);
    }
  }
  rmdirSync(dir);
}
```

现在我们只需要执行下这个脚本，就自动将文件压缩到 `dist/autoUpdate` 文件夹下面去了，

我们可以顺便给他发布到 netlify 上

```typescript
console.log("开始部署到 netlify");

// prettier-ignore
const output = spawn("netlify", [
    "deploy",
    "--dir", zipFolder,
    "--site", "SITE_ID",
    "--auth", "AUTH",
    "--prod",
    "--debug",
  ]);

output.stdout.on("data", function (data) {
  console.log("stdout: " + data.toString());
});

output.stderr.on("data", function (data) {
  console.log("stderr: " + data.toString());
});

output.on("exit", function (code) {
  //  @ts-ignore
  console.log("child process exited with code " + code.toString());
});
```

到现在为止，我们已经完成了自动更新的逻辑。这些代码只是一些思路的分享，实际上到你项目还需要自己调整，比如如何通知渲染进程之类的

# 跨平台发布

你可能已经注意到了，我是在 macos 上进行打包的，有些 node_modules 中的包需要预构建才能在不同的平台上使用，比如 sqlite3,

如果我在 macos 上安装了 sqlite3 那么其他平台的用户，是无法使用的。因为 sqlite3 只有预构建的 darwin 版本，而且还是 arm64 的版本。

因此其他平台是无法使用这个 node_modules 包的，那么如何解决呢？

其实也不是太难，我们只需要提前将 3 个平台的预构建包放到 node_modules 里面即可

那么如何将这些需要预构建的包，提前预构建好，让所有平台都能使用一个 node_modules 包呢？

比如 sqlite3，本身 github 上就提供了对应的预购建版本，

[https://github.com/TryGhost/node-sqlite3/releases/tag/v5.1.6](https://github.com/TryGhost/node-sqlite3/releases/tag/v5.1.6)

我们只需要写个脚本，将这些预购建拉去下来后，塞入到 node_modules 下面即可

```typescript
/**
 * 处理某些包需要预构建才能在多平台上发布
 */

import { join } from "node:path";
import { readFileSync, createWriteStream } from "node:fs";
//  @ts-ignore
import download from "download";

//  此程序，会根据 key 值，去找到对应的 package.json 中的 binary 字段，
//  然后根据包的版本拼接 binary.host + package.version + value
//  然后放到 outputPath 里面
const needPrebuildMap: any = {
  sqlite3: {
    files: [
      "napi-v6-darwin-unknown-arm64.tar.gz",
      "napi-v6-darwin-unknown-x64.tar.gz",
      "napi-v6-win32-unknown-x64.tar.gz",
    ],
    //  相对于 sqlite3 这个包的存放位置
    outputPath: ["lib", "binding"],
  },
};

(async () => {
  const keys = Object.keys(needPrebuildMap);
  for (let i = 0; i < keys.length; i++) {
    let pkg = keys[i];
    const config = needPrebuildMap[pkg];
    const projectFolder = require.resolve(pkg).split("node_modules")[0];
    const pkgFolder = join(projectFolder, "node_modules", pkg);
    const pkgJSON = JSON.parse(
      readFileSync(join(pkgFolder, "package.json"), "utf-8")
    );
    const pkgVersion = pkgJSON.version;
    const hostUrl = pkgJSON.binary.host;
    const outputFolder = join(pkgFolder, ...config.outputPath);

    for (let j = 0; j < config.files.length; j++) {
      // 下载文件
      const downloadURL = `${hostUrl}v${pkgVersion}/${config.files[j]}`;
      const outputPath = join(outputFolder);
      await download(downloadURL, outputPath, {
        extract: true,
      });
      console.log(`下载: ${downloadURL} 成功！`);
    }
  }
})();
```

这个文件在你打包前执行一次就行了。你可以把它和上面其他提到的脚本命令整合成一个命令去执行，这样会方便很多。

# 总结

经过测试，到目前未知自动更新和跨平台都能正常工作，我们在 macos m1 上就打包出来了 win linux 和 macos x64 的软件包。
