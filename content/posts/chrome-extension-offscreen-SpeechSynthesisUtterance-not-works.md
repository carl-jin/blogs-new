---
title: Chrome 插件 chrome.offscreen 中 SpeechSynthesisUtterance 不工作问题
date: 2024-11-06T12:36:23-05:00
lastmod: 2024-11-06T12:36:23-05:00
author: ["CarlJin"]
keywords: 
- 
categories: # 没有分类界面可以不填写
- 
tags: # 标签
- 
description: Chrome 插件 chrome.offscreen 中第一次执行 SpeechSynthesisUtterance 朗读文字不工作，自由第二次执行时才生效问题记录
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
chrome@v130 中的 chrome extension 在调用 `chrome.offscreen.createDocument` 执行 js 文件 `new SpeechSynthesisUtterance` 朗诵不生效问题

该问题只在 130 版本中出现，以此记录。

解决方法如下

```javascript
//  offscreen.js
chrome.runtime.onMessage.addListener(msg => {
  if ('playOffscreenRing' in msg) playAudio(msg.playOffscreenRing);
});


//  ring 如果为文本字符串时，为播放 tts
//  如果为 chrome-extension 开头代表着播放音频
function playAudio(ring) {
  if(ring.startsWith('chrome-extension')){
    const audio = new Audio(ring);
    audio.play();
  }else{
    readTTS(ring)
  }
}

function readTTS(str) {
  //
  //
  //  这里要先初始化一次，什么都不朗读
  //
  //
  window.speechSynthesis.speak(new SpeechSynthesisUtterance(''));

  chrome.runtime.sendMessage({
    type: "FROM_OFFSCREEN",
    data: `朗读 ${str}`
  })

  try{
    return new Promise((resolve) => {
      const utterance = new SpeechSynthesisUtterance(str);
      utterance.lang = "zh-CN";
      utterance.volume = 1;
      utterance.rate = 1;

      window.speechSynthesis.cancel();

      if (window.speechSynthesis.speaking) {
        setTimeout(() => {
          window.speechSynthesis.speak(utterance);
          resolve();
        }, 100);
      } else {
        window.speechSynthesis.speak(utterance);
        resolve();
      }
    });
  }catch (e){
    chrome.runtime.sendMessage({
      type: "FROM_OFFSCREEN",
      data: `朗读错误 ${e.toString()}`
    })
  }
}
```

```javascript
//  background.js
const needDoublePlay = await setupOffscreenDocument("offscreen/index.html");

//
//
//  这里是解决的关键，如果 offscreen 文件是第一次创建，那么就播放两次，这样就能解决第一次播放没声音问题
//
//
if (needDoublePlay) {
  await chrome.runtime.sendMessage({
    playOffscreenRing: source,
  });
  await new Promise((resolve) => setTimeout(resolve, 1500));
}

await chrome.runtime.sendMessage({
  playOffscreenRing: source,
});

async function setupOffscreenDocument(path: string) {
  const offscreenUrl = chrome.runtime.getURL(path);
  const existingContexts = await chrome.runtime.getContexts({
    contextTypes: [chrome.runtime.ContextType.OFFSCREEN_DOCUMENT],
    documentUrls: [offscreenUrl],
  });

  if (existingContexts.length > 0) {
    return false;
  }

  // create offscreen document
  if (creating) {
    await creating;
  } else {
    creating = chrome.offscreen.createDocument({
      url: path,
      reasons: [chrome.offscreen.Reason.AUDIO_PLAYBACK],
      justification: "play an audio",
    });
    await creating;
    creating = null;

    return true;
  }
}

```



