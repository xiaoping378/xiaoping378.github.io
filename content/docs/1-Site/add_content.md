---
tags: ["hugo"]
title: "编写文章注意项"
linkTitle: "编写文章注意项"
weight: 3
description: >
  描述如何添加文章，和主题美化相关的技巧。
---

{{% pageinfo %}}
本章节主要描述日常编写文章的注意事项和docsy主题内的使用技巧。
{{% /pageinfo %}}


## 编写markdown

### 图片路径问题

日常md文件里可以写成`![](/image.png)` ，图片实际要存放到`static`路径下，hugo会自动渲染到站点的根路径`/`上。

这样的话，编辑器编写md就无法预览了，可以使用一个临时目录，把图片和md文件放到同一目录，编写完毕后，再把图片和md文件放置上述合适位置。

### 日常编写注意事项

#### md文件中相互引用内容的路径要记得对应到站点html的路径上

比如此处引用`自动托管markdown`文章，按照md习惯写,虽然编辑器上可以正常打开，但hugo渲染后的页面是不行的。
- `[引用](./actions_pages.md)` 需要改成 `[引用](../actions_pages)`
- `[引用](/content/docs/1-site/actions_pages.md)` 需要改成 `[引用](/docs/1-site/actions_pages)`

知道有这些差异就可以了。日常编写，建议本机启用`hugo server`实时预览，