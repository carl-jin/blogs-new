---
title: "React useImperativeHandle 如何用 typescript 编写"
date: 2023-12-30T14:43:08-05:00
lastmod: 2023-12-30T14:43:08-05:00
author: ["CarlJin"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 
description: ""React useImperativeHandle 都用过，但是如何当它能正确的用 typescript 编写？React 官方文档上并没有做说明。让我们来看个例子"
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

# 例子

下面是一个很简单的一个 antd Input 的封装组件，我们希望它对外暴露一个 `clear` 方法，用于清空输入内容。

```tsx
import { forwardRef, useEffect, useImperativeHandle, useState } from "react";
import { Input, InputProps, Space } from "antd";

export type UserInputPropsType = {
  size: InputProps["size"];
  value: string;
  onChange(val: string): void;
};

//  定义暴露方法的类型
export type UserInputHandler = {
  clear: () => void;
};

//  将类型传递给 forwardRef 的泛型 UserInputHandler
const UserInput = forwardRef<UserInputHandler, UserInputPropsType>(
  ({ size = "middle", value, onChange }: UserInputPropsType, ref) => {
    const [_value, setValue] = useState("");

    //  暴露方法
    useImperativeHandle(
      ref,
      () => ({
        clear() {
          setValue("");
          onChange("");
        },
      }),
      []
    );

    useEffect(() => {
      setValue(value);
    }, [value]);

    return (
      <Space size={12}>
        <Input
          style={{ width: 240 }}
          size={size}
          value={_value}
          onChange={(ev) => {
            setValue(ev.target.value);
            onChange(ev.target.value);
          }}
        />
      </Space>
    );
  }
);

//  这里需要提供下名称
//  不然 typescript 会报错，因为 forwardRef 无法找到组件名称
//  这是个 eslint 的规则 plugin:react/recommended
UserInput.displayName = "UserInput";

export default UserInput;
```

## 使用

来看下其它组件引用这个文件后该如何使用

```tsx
import { ElementRef } from "react";
import { UserInput } from "./UserInput";

const Form = () => {
  //  这里需要用 ElementRef 来定义 UserInput 组件暴露的方法类型
  const UserInputRef = useRef<ElementRef<typeof UserInput>>(null);

  return (
    <div>
      <UserInput {...props} ref={UserInputRef} />
      {/*这里的 clear 已经有类型提示了*/}
      <button onClick={() => UserInputRef.current.clear()}>清空</button>
    </div>
  );
};
```
