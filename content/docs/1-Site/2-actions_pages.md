---
tags: ["hugo"]
title: "自动托管markdown"
linkTitle: "自动托管markdown"
weight: 2
description: >
  描述如何利用github的CI/CD实现自动更新并托管站点。
---

{{% pageinfo %}}
利用github actions和pages实现自动更新托管内容，本站点已实现commit md后，~~自动更新[码云page](https://xiaoping378.gitee.io/)和~~[Github page](https://xiaoping378.github.io/)页面。
{{% /pageinfo %}}

## 项目仓库的名字由来

本源码仓之所以起如此长的名字，完全是因为github pages的工作机制决定的，不然会带个小尾巴：

如果叫 _blog_, github的托管页面的访问地址会是 _xiaoping378.github.io/blog_，这也没什么，但会和hugo的[static机制](/docs/1-site/add_content/#图片路径问题)出现冲突。

