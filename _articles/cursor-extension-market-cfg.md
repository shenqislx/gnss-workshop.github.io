---
layout: article
title: Cursor扩展商店无法正常下载插件
category: 测试开发工具
tags: [Cursor, IDE, 工具配置]
author: GNSS Workshop
---

# Cursor扩展商店无法正常下载插件

有时候我们可能会遇到Cursor IDE的扩展商店（Extensions Marketplace）无法正常下载插件的情况，比如搜索不到插件、比如无法下载安装插件，这时候我们需要检查一下配置文件是否正确。

## 原因

这个问题的原因大概率是网络不通畅，与Cursor的扩展商店的通讯出了问题。
不过我们可以通过修改扩展商店的来源，绕过Cursor的扩展商店，直接从VS Code上下载插件。

## 解决方法

### 1. 找到配置文件

修改Cursor安装目录下的JSON配置文件：`./resources/app/product.json`

### 2. 修改扩展商店配置

关注配置文件中的"extensionsGallery"配置项，它的作用是定义扩展商店相关的 URL 地址，Cursor就是通过这些URL去拉取插件列表、安装插件、获取推荐等。

一言以蔽之，"extensionsGallery" 就是Cursor告诉自己"去哪里找插件商店"的配置。

缺省配置可能如下：
```json
"extensionsGallery": {
    "galleryId": "cursor",
    "serviceUrl": "https://marketplace.cursorapi.com/_apis/public/gallery",
    "itemUrl": "https://marketplace.cursorapi.com/items",
    "resourceUrlTemplate": "https://marketplace.cursorapi.com/{publisher}/{name}/{version}/{path}",
    "controlUrl": "https://api2.cursor.sh/extensions-control",
    "recommendationsUrl": "",
    "nlsBaseUrl": "",
    "publisherUrl": ""
}
```

我们需要修改 serviceUrl 和 recommendationsUrl 两个参数：

```json
"extensionsGallery": {
    ...
    "serviceUrl": "https://marketplace.visualstudio.com/_apis/public/gallery",
    "recommendationsUrl": "https://{publisher}.vscode-unpkg.net/{publisher}/{name}/{version}/{path}",
    ...
}
```

### 3. 重启应用

重启IDE，就可以正常下载插件了。

**注意**：如果你要在内网搭建自己的扩展商店，就可以把 "extensionsGallery" 改成自己的服务地址。这样开发者就可以在内网安装/搜索插件，而不依赖公网Marketplace。