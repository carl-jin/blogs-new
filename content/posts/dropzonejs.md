---
title: "Dropzonejs 6.0.0 如果在已有的 Form 中添加 Drop2Upload"
date: 2023-11-22T10:20:23-05:00
lastmod: 2023-11-22T10:20:23-05:00
author: ["CarlJin"]
keywords:
  -
categories: # 没有分类界面可以不填写
  -
tags: # 标签
  -
description: "如何将 dropzonejs 添加到网站中已有的 form 表单中？dropzone 官方文档写的是挺漂亮的，但是也就是停留在漂亮，一点也不实用，甚至还要去扒源代码。本文将想你介绍解决方案"
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

网站使用 Wordpress 搭建，现在要在一个 wpcf7 的插件表单中添加一个拖拽上传功能。目前只找到 dropzonejs 比较符合需求。但是 dropzonejs 文档写的过于简陋，如何将文件上传功能嵌入到 form 并且在 submit 时一并提交过去（不是异步上传文件），这在文档上中的 options 和 methods 根本找不到解决办法。经过折腾后找到以下解决方案。

## 快捷通道

如果你使用的也是 wpcf7 ，那么直接使用 [Drag and Drop Multiple File Upload – Contact Form 7](https://wordpress.org/plugins/drag-and-drop-multiple-file-upload-contact-form-7/) 这个插件吧，下面的内容都不需要看了

## 1. 加载 dropzone

```typescript
import Dropzone from "dropzone";
import "dropzone/dist/dropzone.css";
```

## 2. 在 form 中添加拖拽上传区域

下面是在原油的 file 字段的下方添加一个 `.dropzone-box` 节点，用于显示 dropzone UI

```typescript
const $fileLabel = $(".wpcf7-file").parents("label");
$fileLabel.hide();
const $title = $fileLabel.text().trim();

$fileLabel.parent().append(
  $(`
    <div class="dropzone-box">
      <p>${$title}</p>
      <div class="dropzone">
        <div class="upload-hint">
          <div class="icon"></div>
          <div>Drag & drop reference images here or <span class="highlight">Browse</span></div>
        </div>
        <div class="dropzone-preview"></div>
      </div>
    </div>
  `)
);
```

## 3. 初始化 dropzone

```typescript
//  因为不希望它影响到原有的 form
//  所以这里选择的 DOM 是 .dropzone 而不是一个 form
let myDropzone = new Dropzone(".dropzone", {
  //  因为我们没有选择 form ，必须提供 url，这里随便输入一个字符串就行了
  //  因为我们会重写 submitRequest 方法，所以这个地址并不会被访问
  url: "/",
  //  这里要设置未 true，这样才会在点击上传文件时能显示上传成功动画
  autoProcessQueue: true,
  uploadMultiple: true,
  parallelUploads: 100,
  maxFiles: 100,
  addRemoveLinks: true,
  //  这里是第 2步中生成的预览文件存放的 DOM
  previewsContainer: ".dropzone-preview",
  //  设置可点击上传的 DOM
  clickable: ".dropzone",
});

//  默认情况下，在上传文件时 dropzonejs 会将文件提交到 url 参数指定的地址上去
//  但是我们这里的需求是希望它跟 wpcf 一起发送
//  因此这里当文件发送时直接全部设置为成功
myDropzone.submitRequest = (_, __, files) => {
  for (let file of files) {
    file.status = "success";
    myDropzone.emit("success", file, "success");
    myDropzone.emit("complete", file);
  }
};
```

## 4. 监听文件添加和删除

dropzonejs 添加文件和删除时，我们需要将文件保存下来，以便在 wpcf 提交时同时提交过去

```typescript
let files: File[] = [];
myDropzone.on("addedfile", (file) => {
  files.push(file);
  console.log(`File added: ${file.name}`);
});

myDropzone.on("removedfile", (file) => {
  files = files.filter((_file) => _file !== file);
  console.log(`File removed: ${file.name}`);
});
```

## 5. 拦截 wpcf submit 并且提交 files

```typescript
//  手动处理 form 提交
$(".wpcf7-submit").on("click", (ev) => {
  ev.preventDefault();
  ev.stopPropagation();

  (async () => {
    //  这里调用的是 wpcf7 提供的全局方法
    //  通过阅读其源代码，发现可以通过 submit 的第二个参数提供 submitter 来额外添加字段
    window["wpcf7"].submit($(".wpcf7-form").get(0), {
      submitter: {
        name: "files",
        value: await files.map(async file=> await fileToBuffer(file))
      },
    });
  })();
});

async function fileToBuffer(file: File) {
  return new Promise<any>((res) => {
    var reader = new FileReader();

    reader.onload = function (event) {
      //  @ts-ignore
      res(reader.result);
    };

    reader.readAsArrayBuffer(file);
  });
}
```
