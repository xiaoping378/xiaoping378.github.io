---
date: 2022-01-24
title: "带图片预览的题目"
linkTitle: "图片博客"
description: "本文展示带图片预览的博客写法"
author: xiaoping378
resources:
- src: "**.{png,jpg}"
  title: "Image #:counter"
  params:
    byline: "Pics: xiaoping378 / CC-BY-CA"
---

**This is a typical blog post that includes images.**

The front matter specifies the date of the blog post, its title, a short description that will be displayed on the blog landing page, and its author.

## Including images

Here's an image (`featured-sunset-get.png`) that includes a byline and a caption.

{{< imgproc sunset Fill "600x300" >}}
图片拉伸效果
{{< /imgproc >}}

{{< imgproc index-2022-01-24 Fill "600x300" >}}
图片拉伸效果
{{< /imgproc >}}


The front matter of this post specifies properties to be assigned to all image resources:

```
resources:
- src: "**.{png,jpg}"
  title: "Image #:counter"
  params:
    byline: "Photo: Riona MacNamara / CC-BY-CA"
```

To include the image in a page, specify its details like this:

```
{{< imgproc sunset Fill "600x300" >}}
Fetch and scale an image in the upcoming Hugo 0.43.
{{< /imgproc >}}
```

The image will be rendered at the size and byline specified in the front matter.