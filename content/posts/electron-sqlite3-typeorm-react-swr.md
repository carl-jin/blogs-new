---
title: "在Electron 中 拥抱 Type-safe 的 Sqlite3"
date: 2023-03-28T11:27:36-04:00
lastmod: 2023-03-28T11:27:36-04:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  - electron
  - typeORM
description: "如何在 Electron 优雅的使用支持 Type-safe 的 Sqlite3 数据库？本文将与你分享这个探究过程以及最终实现的效果"
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
  image: "https://miro.medium.com/v2/resize:fit:700/1*UfVpXivjrf2WiSl1pZtYRQ.png" #图片路径例如：posts/tech/123/123.png
  zoom: # 图片大小，例如填写 50% 表示原图像的一半大小
  caption: "" #图片底部描述
  alt: ""
  relative: false
---

## 应用场景

最近需要重写一个老的 Electron 项目，其中涉及到了 Sqlite 的使用，感觉之前的写法比较别扭，故此想着这次重写时让它对于 Sqlite 的操作完全拥抱 Typescript.

### 先来看看之前的 Sqlite 操作方式

> 假设我们要添加一个用户

```tsx
//  渲染进程
function AddNewUserButton() {
  async function handleAddNewUser() {
    const user = await window.electron.ipcRenderer.invoke("addNewUser", {
      name: "张三",
      email: "zhangsan@gmail.com",
      password: "123",
      authentication: "",
    });

    console.log(user.id);
  }
  return <button onClick={handleAddNewUser}>点击添加新用户</button>;
}
```

```tsx
//  主进程
import { ipcMain } from "electron";
import sqlite from "sqlite3";

const database = new sqlite.Database(
  path.join(app.getPath("userData"), "database.db")
);

ipcMain.handle("addNewUser", (_, userInfo) => {
  return this.execute(
    `REPLACE INTO accounts (id, name, email, password, authentication, pageLink, cookies, groupList) VALUES ("${id}", "${escape(
      name
    )}", "${escape(email)}", "${escape(
      password
    )}", "${authentication}", "${escape(pageLink)}", "${escape(
      cookies
    )}", "${groupList}")`
  );
});
```

**注意这仅仅是一个数据库操作的代码，实际项目有几十个类似的操作**

先来看下这个老项目这样操作 sqlite 的问题

1. 没有 typescript 支持，因此渲染进程里面的 invoke 调用和 主进程里面的 sql 执行代码都没有参数提示和约束，也就是没有 type-safe. 都 2023 年了，这样编写方式很别扭，现在年纪大了**记忆力**也下降了加上要维护很多项目，导致每次改起逻辑来都要重新查看下数据库，确保字段正确并且还要琢磨有没有边界情况，😫 头发都掉光光, 每次因为边界问题没处理好，导致线上程序出现 bug 搞得也是心惊胆战。
2. Node js 项目中直接使用 SQL 语句，说不上来的别扭。阅读起来麻烦
3. invoke 和 handle 没有类型约束与提示，渲染进程该传什么参数，大部分是要靠猜。（API 文档？你都全栈开发哪里来的文档？）
4. 数据库结构不清晰，要想了解数据库结构必须要用数据库软件看，或者读生成数据库的 SQL 语句。

## 理想情况？

理想的情况应该满足以下几个条件

1. 完全的 typescript 支持，每个方法都提供了具体的参数和返回值的类型，这样能提高开发效率和低级错误
2. 数据库应该有个 schema 文件用来描述数据库中每个表的结构
3. 使用第三方库操作数据库，而不是手写 SQL
4. 主进程和渲染进程中的数据类型应该统一，不需要再额外维护一套类型
5. 给 react-swr 也加上数据库操作的类型支持

## 完成后的效果

> 下面几张图是完成后的效果

React 中调用数据库操作接口的方法，有明确的类型提示，告诉你它接收什么，返回什么
![https://i.imgur.com/vSZNA8a.png](https://i.imgur.com/vSZNA8a.png)

参数传递错误时会有直接的报错
![https://i.imgur.com/149Uuzs.png](https://i.imgur.com/149Uuzs.png)

也可以自动判断返回值
![https://i.imgur.com/m6gXFai.png](https://i.imgur.com/m6gXFai.png)

swr 也有类型支持
![https://i.imgur.com/vgEois5.png](https://i.imgur.com/vgEois5.png)

swr 参数传递错误时也有提示
![https://i.imgur.com/px3qsu8.png](https://i.imgur.com/px3qsu8.png)

整个 swr 方法都有提示！
![https://i.imgur.com/pWvbCTi.png](https://i.imgur.com/pWvbCTi.png)

数据库操作使用 typeORM，从未有过的简洁（相对于写 SQL)

并且每个方法都有类型支持
![https://i.imgur.com/nB1r9vl.png](https://i.imgur.com/nB1r9vl.png)

## 技术选型

> 学轮子的过程远比造轮子来的轻松和愉快

为了完成上述的理想情况，运用到了以下的技术

1. [vite-electron-builder](https://github.com/cawa-93/vite-electron-builder) electron 项目模版，它里面内置了 Vue，但是由于我要用 React 便修改成了 React 版本。用什么模版其实影响不大，都是那几个文件，main,renderer,preload. 选这个模版主要是因为比较熟悉 vite，而且 vite 是真的快！
2. sqlite3 用作数据库，因为项目本身不大 sqlite3 是最好的选择了
3. [typeORM](https://typeorm.io/) ORM 系统，为什么选择它？而不是 sequelize 或者 prisma？首先 sequelize 和 prisma 都尝试过最后才选择的 typeORM（虽然官方文档有点糟糕）， sequelize type 支持不是太行放弃了，prisma 对 electron 的兼容不行，现在 GitHub 上相关的 issue 还没解决，而且 prisma 的 schema 用的不知道是啥的描述文件(.prisma) 风格实在接受不来。反观 typeORM，完全拥抱的 ts 语法和 electron 支持，目前发现最合适的 ORM
4. [swr](https://swr.vercel.app/) 用接口请求和缓存，用过的都说好。

## 代码实现

> 这篇文章不是面向初学者的，你既然读到这里就已经假设你至少了解和掌握以下技术，typescript, electron, vite, react, typeORM, sqlite3, swr.

### 数据库结构

我们先来看看之前最头疼的数据库每张表的结构问题怎么解决。之前都是写 SQL 生成表，这样阅读起来比较麻烦。

现在我们使用 TypeORM 后，将对应的表转换成对应的 entity 文件就行了

> （tip：你可以把之前的 SQL 语句给 ChatGPT 让它帮你转换成 entity 类型就行了）

我们在主进程的文件夹下面建立下面这个文件，这里只用一个 AccountTags 表来做演示。
如果你数据库中有其他的表，也是类似的

```typescript
//  main/src/db/entity/AccountTags.ts
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm";

@Entity()
export class AccountTags {
  @PrimaryGeneratedColumn()
  id: string;

  @Column({ type: "text" })
  title: string;

  @Column({ type: "text" })
  color: string;

  @Column({ type: "integer" })
  createdAt: number;
}
```

建立完上面这个 entity 之后，我们现在就可以使用 AccountTags 这个类型了，比如下面

```typescript
import { AccountTags } from "@db/entity/AccountTags.ts";

//  简单又方便！
const tag: Omit<AccountTags, "id" | "createdAt"> = {
  title: "标签1",
  color: "#f20",
};
```

### 数据库操作类

现在我们可以使用 TypeORM 来操作 sqlite，因此我们将每张表操作的逻辑提取成一个 Operator

```typescript
//  main/src/db/operators/AccountTags.ts
import { DataSource, Repository } from "typeorm";
import { AccountTags } from "@/db/entity/AccountTags";
import { getCurrentTimestamp } from "@/utils/utils";

export class AccountTagsOperator {
  Repository: Repository<AccountTags>;
  AppDataSource: DataSource;

  constructor(source: DataSource) {
    this.AppDataSource = source;
    this.Repository = this.AppDataSource.getRepository(AccountTags);
  }

  /**
   * 获取所有 tags
   */
  async getAccountTags(): Promise<AccountTags[]> {
    return new Promise((res) => {
      setTimeout(async () => {
        res(
          await this.Repository.find({
            order: {
              createdAt: "DESC",
            },
          })
        );
      }, 2000);
    });
  }

  /**
   * 添加一个 tag
   * @param tag
   */
  async addAccountTag(
    tag: Pick<AccountTags, "title" | "color">
  ): Promise<AccountTags> {
    const newTag = this.Repository.create({
      ...tag,
      createdAt: getCurrentTimestamp(),
    });

    return this.Repository.save(newTag);
  }
}
```

由于之前创建了对应的 entity 这里可以直接用它的类型进行约束，我们只需要将每个操作数据库的逻辑，分装成这个 class 上的 method 就行了

调用起来是这样的 (这只是个例子，实际场景中这样使用还是太麻烦)

```typescript
const A = new AccountTagsOperator(DataSource);
A.getAccountTags().then((tags) => {
  console.log(tags);
});
```

同时我们在 `main/src/db/operators/index.ts` 中将所有的 operators 导出，方便后续其他地方引用

```typescript
//  main/src/db/operators/index.ts
import { AccountTagsOperator } from "@/db/operators/AccountTags";

const operators = {
  AccountTags: AccountTagsOperator,
};

export type OperatorsType = {
  [K in keyof typeof operators]: Omit<
    InstanceType<(typeof operators)[K]>,
    "Repository" | "AppDataSource"
  >;
};

export { operators };
```

注意我们这里定义了一个 `OperatorsType`,

它类似与这样的一个类型 `{ AccountTags: { getAccountTags(); addAccountTag()  } `

这样我们只需要个变量断言成这个类型，那么就会有对应的语法提示了

比如下面这个例子

```typescript
const a: OperatorsType = {};

//  将会有完整的类型支持
a.AccountTags.getAccountTags();
```

### 抽离出 DB 类

由于所有的数据库操作，都应该和主进程的逻辑代码分开，所以我们建立一个 DBServer 这个类，来提供 DB 的操作

```typescript
//  main/src/db/DBServer.ts
import { DataSource } from "typeorm";
import { ipcMain } from "electron";

import { AccountTags } from "./entity/AccountTags";
import { operators as operatorsMap, OperatorsType } from "@/db/operators";
import { get } from "lodash";
import { formatDurationToS } from "@/utils/utils";

export class DBServer {
  AppDataSource: DataSource;
  operators: OperatorsType;

  init(path: string): Promise<void> {
    //  定义 typeORM 的 DataSource
    this.AppDataSource = new DataSource({
      type: "sqlite",
      database: path,
      entities: [AccountTags],
      synchronize: true,
      logging: false,
    });

    return new Promise((res, rej) => {
      this.AppDataSource.initialize()
        .then(async () => {
          this.initOperators();
          res();
        })
        .catch((error) => rej(error));
    });
  }

  public close() {
    this.AppDataSource.destroy().then();
  }

  //  初始化 operators
  //  这样我们才能在后续的代码中这样使用 this.operators.AccountTags.getAccountTags()
  private initOperators() {
    this.operators.AccountTags = new operatorsMap.AccountTags(
      this.AppDataSource
    );
  }
}
```

然后在主进程中调用在就行了，非常的干净

```typescript
//  main.js
import { DBServer as _DBServer } from "./db/DBServer";

const DBServer = new _DBServer();

app.whenReady().then(() => {
  return DBServer.init("./db.sqlite").then(() => {
    //   todos omething
  });
});
```

在主进程里面，要操作数据库，可以这样写

```typescript
DBServer.operators.AccountTags.getAccountTags();
```

### DB 类，提供 IPC 消息处理

由于最终我们的业务还是要让渲染进程掉用接口来操作数据库，也就是使用 IPC 来通讯，因此我们需要在 DB 类中监听一个消息，来处理渲染进程中的数据库操作请求

我们来修改下刚刚的代码

```typescript
//  main/src/db/DBServer.ts
import { DataSource } from "typeorm";
import { ipcMain } from "electron";

import { AccountTags } from "./entity/AccountTags";
import { operators as operatorsMap, OperatorsType } from "@/db/operators";
import { get } from "lodash";
import { formatDurationToS } from "@/utils/utils";

const DATABASE_OPERATE_MESSAGE_NAME = "database_operate_message";

export class DBServer {
  AppDataSource: DataSource;
  operators: OperatorsType;

  init(path: string): Promise<void> {
    //  定义 typeORM 的 DataSource
    this.AppDataSource = new DataSource({
      type: "sqlite",
      database: path,
      entities: [AccountTags],
      synchronize: true,
      logging: false,
    });

    return new Promise((res, rej) => {
      this.AppDataSource.initialize()
        .then(async () => {
          this.initOperators();
          this.bindMessage();
          res();
        })
        .catch((error) => rej(error));
    });
  }

  public close() {
    this.AppDataSource.destroy().then();
  }

  //  初始化 operators
  //  这样我们才能在后续的代码中这样使用 this.operators.AccountTags.getAccountTags()
  private initOperators() {
    this.operators.AccountTags = new operatorsMap.AccountTags(
      this.AppDataSource
    );
  }

  private bindMessage() {
    ipcMain.handle(DATABASE_OPERATE_MESSAGE_NAME, async (_, path, ...args) => {
      const start = Date.now();
      try {
        const [entity, methodName] = path.split(".");
        const operator = get(this.operators, path, false);
        if (!operator) throw new Error(`${path} 不存在！`);
        //  @ts-ignore
        const data = await this.operators[entity][methodName](...args);
        const end = Date.now();

        return {
          type: "success",
          error: undefined,
          result: data,
          duration: formatDurationToS(end - start),
          operator: path,
          args: args,
        };
      } catch (e) {
        const end = Date.now();

        return {
          type: "error",
          // @ts-ignore
          error: e.toString(),
          result: undefined,
          duration: formatDurationToS(end - start),
          operator: path,
          args: args,
        };
      }
    });
  }
}
```

上面代码中，监听了一个个消息 `DATABASE_OPERATE_MESSAGE_NAME` ("database_operate_message"), 这样只要给 invoke 参数传递对应的参数，就可以用来操作数据库，比如

```typescript
const tags = await invoke(
  "database_operate_message",
  "AccountTags.getAccountTags"
);

//  如果要传递参数就这样
await invoke("database_operate_message", "AccountTags.addAccountTag", {
  title: "title",
  color: "#f20",
});
```

### 将 invoke 通过 preload 提供给渲染进程

```typescript
//  preload.ts
import { ipcRenderer } from "electron";

async function dbMessage(messageName: string, ...args) {
  return await ipcRenderer.invoke(messageName, ...args);
}

export { dbMessage };
```

> ⚠️ 上面的 preload 写法是通过 `unplugin-auto-expose` 这个库实现的，如果你用的是 vite-electron-builder 那么它自带的就有
> 这样写完后在渲染进程里面只需要这样调用就行
>
> import { dbMessage } from '#preload'

### 渲染进程如何实现接口调用?

上面写了一大堆的代码，都是在主进程里面，那么回到渲染进程，我们现在该如何实现 `window.db.AccountTags.getAccountTags()` 这样的接口调用呢？

先列出几个不可行的方法

1. ❌ 直接引用主进程的 operators/index.ts 文件
2. ❌ 通过 preload 导出
3. ⚠️ 单独在 renderer 里面单独写一个 operators/index.ts，😓😓 这岂不是要维护两个数据操作对象和类型，为何要为难自己呢，不干。

介于没有啥好的解决办法，因此采用下面比较笨的方法，实现 window.db 方法的初始化

{{< mermaid >}}
sequenceDiagram
渲染进程->>+主进程: 通过 invoke 请求 Operators 数据结构
主进程-->>-渲染进程: 返回 Operators 对象结构（仅仅是结构）
渲染进程->>渲染进程: 根据 Operators 对象，生成对应的 window.db 对象
渲染进程->>渲染进程: 初始化 React
渲染进程->>+主进程: React 组件中 通过 window.db 调用 invoke 发送消息
主进程->>主进程: 根据 invoke 中的 operators 信息，操作数据库
主进程-->>-渲染进程: 返回数据
{{</ mermaid >}}

### 代码实现

我们首先在 `main/src/db/DBServer.ts` 中监听一个 IPC 消息

```typescript
//  main/src/db/DBServer.ts
export const GET_DATABASE_OPERATE_STRUCTURE_NAME =
  "database_operate_structure_message";

export class DBServer {
  // ...

  private bindMessage() {
    // ...

    /**
     * 将 operators 的数据类型对象返回给前端
     * 前端根据这个对象进行转换为
     * window.db.Account.getAccountById = function(...args){
     *  invoke("Account.getAccountById", ...args)
     * }
     */
    ipcMain.handle(GET_DATABASE_OPERATE_STRUCTURE_NAME, (async) => {
      const structureObj: any = {};

      for (let entity in this.operators) {
        //  @ts-ignore
        const instance = this.operators[entity] as any;
        const methods: any = {};

        const methodNames = Object.getOwnPropertyNames(
          Object.getPrototypeOf(instance)
        ).filter(
          (propName) =>
            typeof instance[propName] === "function" &&
            !["constructor"].includes(propName)
        );

        methodNames.map((method) => {
          methods[method] = true;
        });

        structureObj[entity] = methods;
      }

      return structureObj;
    });
  }
}
```

上面代码中，监听了一个个消息 `GET_DATABASE_OPERATE_STRUCTURE_NAME` ("database_operate_structure_message"), 这样只要给 invoke 参数传递对应的参数，就可以获取当前的 operators 结构

```typescript
const structure = await invoke("database_operate_structure_message");
console.log(structure);
//  { AccountTags: { getAccountTags: true, addAccountTag: true  } }
```

现在我们只需要根据这个返回的 structure 来生成 window.db 这个对象即可，我们在渲染进程里面封装一个 `InjectDBClient.ts`

同时为了解决 React 18 的 StrictMode 带来的 2 次 useEffect 调用，我们同时包裹了以下 `p-cancelable`

具体逻辑看下面代码注释

```typescript
//  renderer/InjectDBClient.ts

import { dbMessage } from "#preload";
import { ErrorMessage } from "@/utils/message";
import PCancelable from "p-cancelable";

export async function injectDBClient(): Promise<void> {
  //  首先通过 dbMessage 发送 IPC 消息给 DBServer
  //  dbMessage 由 preload 提供
  const structure = await dbMessage("database_operate_structure_message");
  window.db = {};

  //  根据 structure 生成 window.db 对象
  //  没有啥技巧，非常的傻瓜
  Object.entries(structure).forEach(([key, value]) => {
    window.db[key] = {};
    Object.keys(value).forEach((methodName) => {
      //  生成对应的方法，比如 window.db.AccountTags.getAccountTags
      window.db[key][methodName] = function (...args) {
        //  这里返回 PCancelable，这样在 useEffect 的销毁副作用方法中，就可以取消请求了
        return new PCancelable(async (resolve, reject, onCancel) => {
          let canceled = false;

          try {
            onCancel.shouldReject = false;
            onCancel(() => {
              canceled = true;
            });
            //  发送 IPC 给 DbServer.ts 请求操作数据库
            const res = await dbMessage(
              "database_operate_message",
              `${key}.${methodName}`,
              ...args
            );

            if (canceled) {
              return;
            }

            if (res.type === "success") {
              //  请求结束返回即可
              return resolve(res.result);
            }

            if (res.type === "error") {
              throw new Error(res.error);
            }

            throw new Error("未知的数据类型");
          } catch (e: any) {
            console.error(e);
            ErrorMessage(e.toString());

            return reject(e);
          }
        });
      };
    });
  });
}
```

这样我们在渲染进程初始化 React 之前调用下即可

```tsx
injectDBClient().then(() => {
  const container = window.document.getElementById("root") as HTMLDivElement;
  const root = createRoot(container);
  root.render(<App />);
});

//  这样在 App 这个组件中就可以调用
//  window.db.AccountsTag.getAccountTags() 了
```

### SWR 支持

既然现在已经有了 window.db 这个操作数据库的对象，那么我们就顺便把它当成 fetcher 来实现下 SWR Provider

```tsx
import { SWRConfig } from "swr";
import { ReactNode } from "react";
import { get } from "lodash";
import { ErrorMessage } from "@/utils/message";

export function SWRProvider(props: { children: ReactNode }) {
  async function fetcher(resource: unknown[]) {
    try {
      const path = resource[0] as string;
      const args = resource.slice(1);

      const method = get(window.db, path, false) as any;
      if (method) {
        return await method(...args);
      } else {
        return Promise.reject(new Error(`未知的方法调用 ${path}`));
      }
    } catch (e) {
      return Promise.reject(e);
    }
  }

  return (
    <SWRConfig
      value={{
        fetcher,
        shouldRetryOnError: false,
        onError: (error) => {
          console.error(error);
          ErrorMessage(error.toString());
        },
      }}
    >
      {props.children}
    </SWRConfig>
  );
}
```

然后我们将 Provider 包裹着 App 组件

```tsx
injectDBClient().then(() => {
  const container = window.document.getElementById("root") as HTMLDivElement;
  const root = createRoot(container);
  root.render(
    <SWRProvider>
      <App />
    </SWRProvider>
  );
});
```

现在你可以在 App 组件中这样写了

```tsx
function App() {
  const { data, isLoading, mute } = useSWR(["AccountsTag.getAccountsTag"]);

  if (!data) return <>loading</>;

  return <span>TagsCount: {data.length}</span>;
}
```

### Type 支持！

你可能已经注意到了，渲染进程中使用的 window.db 和 swr 的调用并没有任何的类型支持。

现在让我们给他加上类型支持吧！

首先我们需要明确一点，直接调用 main 文件夹里面的类型文件**在某些情况下**是行不通的

比如下面这种情况

因为我用的是 vite-electron-builder 它将 main, preload, renderer 拆成了不同的文件夹，

有不同的 tsconfig.json 和 vite.config.js ，也可以理解为类似于 pnpm 的 monorepo 类型项目，
因此直接引用其他文件夹的类型，可能会因为 alias 导致路径错误；

虽然可以在 `main/src/db/operators/index.ts` 中使用相对路径引用其他文件，但是这就很不合适，毕竟它是一个坑，后期开发时，一旦不小心用了 alias 路径，就会报错，然后纠结半天不知道咋回事

因此下面这种调用是虽然是可以行的，但是不推荐 🙅

```typescript
//  renderer
import type { OperatorsType } from "main/src/db/operators/index.ts";
```

那么如何解决呢，其实也很简单，我们直接将对应的 types 通过 tsc 和 tsc-alias 命令将类型文件打包出来，然后在 renderer 中引用不就 ok 了嘛.

我们先在主进程文件夹里面建一个文件 `main/src/db/exportTypes.ts`， 它主要就是负责导出类型，让其他非主进程的目录可以放心且安全的引用类型。

内容如下

```typescript
//  main/src/db/exportTypes.ts
export type { AccountTags } from "./entity/AccountTags";
export type { OperatorsType } from "./operators/index";
```

然后建立一个对应的 tsconfig.json 用于打包用

```json
{
  "compilerOptions": {
    "module": "esnext",
    "target": "esnext",
    "sourceMap": false,
    "moduleResolution": "Node",
    "skipLibCheck": true,
    "strict": true,
    "isolatedModules": true,
    "types": ["node"],
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    },
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "strictPropertyInitialization": false,
    "outDir": "./dist/dbType"
  },
  "extends": "./tsconfig.json",
  "include": ["src/db/exportTypes.ts"]
}
```

然后在 package.json 中添加一个命令

```json
{
  "scripts": {
    "build:DBTypes": "tsc -p ./packages/main/tsconfig.dbTypes.json -emitDeclarationOnly --declaration && tsc-alias -p ./packages/main/tsconfig.dbTypes.json"
  }
}
```

现在我们只需要运行下命令，就可以将 database 类型打包处理

```shell
$ pnpm run build:DBTypes
```

然后我们顺便在 renderer 的 tsconfig.json 中添加一个 alias

```json
{
  "compilerOptions": {
    "paths": {
      "@db/*": ["../main/dist/dbType/db/*"]
    }
  }
}
```

这样我们就可以在渲染进程中这样引用 type 了

```typescript
import { OperatorsType } from "@db/exportTypes";
```

### 加上自动编译

现在我们必须手动执行，才能打包出来 types 文件给 renderer 使用。

这显然不合理，它应该自动监听更改然后重新打包

```shell
$ pnpm run build:DBTypes
```

我们只需要在 vite-electron-builder 提供的 watch.ts 文件中，添加以下代码即可

```typescript
function setupMainPackageWatcher({ resolvedUrls }) {
  // ...

  return build({
    // ...
    plugins: [
      {
        name: "reload-app-on-main-package-change",
        writeBundle() {
          // ...
          /**
           * 更新 db type
           */
          spawn("pnpm", ["run", "build:DBTypes"], {
            stdio: "inherit",
          });
        },
      },
    ],
  });
}
```

这样当我们修改主进程的代码，它就会自动打包出 types

### 完善 window.db 的 types

我们在 renderer/src/global.d.ts 中添加一个 window.db 的类型

```typescript
import { OperatorsType } from "@db/exportTypes";

export declare global {
  interface Window {
    db: OperatorsType;
  }
}
```

🎉🎉🎉 现在在 renderer 中调用 window.db 已经有了完全的类型支持了！

### 完善 swr 的 types

现在虽然导出的有 OperatorsType，我们也可以通过下面这种方式给 useSWR 设置类型，比如

```tsx
import { OperatorsType } from "@db/exportTypes";

function App() {
  //  类型定义太麻烦！
  const { data } = useSWR<
    Awaited<ReturnType<OperatorsType["AccountTags"]["getAccountTags"]>>
  >(["AccountTags.getAccountTags"]);
}
```

这样约束类型，还不如不写，咋还越来越麻烦呢？我们来看看怎么优化它

我们新建一个 hooks，用于包裹 SWR, 同时建立两个泛型用于 Args 和 Return 的类型约束

```typescript
//  main/src/db/exportTypes.ts
export type { AccountTags } from "./entity/AccountTags";
import type { OperatorsType } from "./operators/index";

//  用于定义 数据库操作方法的 返回类型
type DBReturn<
  Entity extends keyof OperatorsType,
  Method extends keyof OperatorsType[Entity]
> = OperatorsType[Entity][Method] extends (...args: any[]) => Promise<infer R>
  ? R
  : never;

//  用于定义 数据库操作方法的 参数类型
type DBArgs<
  Entity extends keyof OperatorsType,
  Method extends keyof OperatorsType[Entity]
> = OperatorsType[Entity][Method] extends (...args: infer Args) => any
  ? Args
  : never;

export type { OperatorsType, DBReturn, DBArgs };
```

```typescript
//  renderer/hooks/useDBSWR
import { OperatorsType } from "@db/operators";
import { DBArgs, DBReturn } from "@db/exportTypes";
import useSWR from "swr";

export function useDBSWR<
  Entity extends keyof OperatorsType,
  Method extends keyof OperatorsType[Entity]
>(resource: [string, ...DBArgs<Entity, Method>], ...args) {
  return useSWR<
    DBReturn<Entity, Method>,
    undefined,
    [string, ...DBArgs<Entity, Method>]
  >(
    resource,
    //  @ts-ignore
    ...args
  );
}
```

这样我们要想使用 swr 并且有类型支持，只需要这样调用就行了

```tsx
function App() {
  //  😯 精简多了，而且有类型提示
  const { data, isLoading, mutate } = useDBSWR<"AccountTags", "getAccountTags">(
    ["AccountTags.getAccountTags"]
  );
}
```

到此为止，我们已经实现了完全的 sqlite 数据库操作的 typescript 支持！

## 总结

必要，太必要了，现在写起来有 typescript 支持和类型安全的帮助，写起来快，用起来也放心。保住了秀发 :)
