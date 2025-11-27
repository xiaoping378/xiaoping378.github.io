---
tags: ["hugo"]
title: "编写文章技巧"
linkTitle: "编写文章技巧"
weight: 3
description: >
  描述在hugo的Docsy主题下如何编写文章和添加图片路径处理等相关的技巧。
---

{{% pageinfo %}}
本章节主要描述日常编写文章的注意事项和docsy主题内的使用技巧。
{{% /pageinfo %}}


## 编写markdown

### 图片路径问题

习惯md文件中的图片路径写成`![](/images/file.png)` ，但hugo中图片实际要存放到`/static/images/`路径下，hugo会自动渲染到站点的根路径`/`上。

这样的话，无法在编辑器中预览md了，有两种玩法如下：

- 可以使用一个临时目录，把图片和md文件放到同一目录，编写完毕后，再把图片和md文件放置上述合适的目录位置上。
- 日常一直启用`hugo server`编写文章，放弃编辑器中预览md，后面还会有别的坑....

推荐后者，本人日常使用VSCode编写md，和代码开发同款，这里推荐安装`mushan.vscode-paste-image`扩展，再额外如下配置：
```json
    "pasteImage.namePrefix": "${currentFileNameWithoutExt}-",
    "pasteImage.path": "${projectRoot}/static/images/",
    "pasteImage.basePath": "${projectRoot}",
    "pasteImage.insertPattern": "${imageSyntaxPrefix}/images/${imageFileName}${imageSyntaxSuffix}"
```

日常编写文章，`Alt+A`截图，`Ctrl+Alt+A`粘贴到md文件，和hugo的配合，完美...

### SEO优化关注点
日常文章编写，如下部分要精准描述，Google搜索引擎推荐使用`description`的meta标签告诉它页面内容的，Docsy主题会自动把红框部分填充到`layouts/partials/head.html`中。

![](/images/add_content-2022-01-25-09-38-48.png)

hugo解析编译后，html页面会如下呈现：

![](/images/add_content-2022-01-25-09-37-12.png)

### 日常编写注意事项

#### md文件中相互引用内容的路径要记得对应到站点html的路径上

比如此处引用`自动托管markdown`文章，按照md习惯写,虽然编辑器上可以正常打开，但hugo渲染后的页面是404的。
- `[引用](./actions_pages.md)` 需要改成 `[引用](../actions_pages)`
- `[引用](/content/docs/1-site/actions_pages.md)` 需要改成 `[引用](/docs/1-site/actions_pages)`



不定时更新注意事项。日常编写，建议本机启用`hugo server`实时预览，

#### 文章weight的使用

文章开头的`weight`部分决定了目录中的排序，推荐新开的系列文章，从两位数开始递增，比如`30`, 以后老系列有更新的时候，免去批量修改调整排序的麻烦。