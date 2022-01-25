---
tags: ["hugo"]
title: "站点搭建方法"
linkTitle: "站点搭建方法"
weight: 1
description: >
  介绍本站点搭建的方法.
---

{{% pageinfo %}}
主要涉及到hugo和docsy主题的使用。
{{% /pageinfo %}}

## hugo介绍

hugo是spf13的开源作品，目前任职于google，对他最早的印象是使用他的vim-spf13配置，后来又接触到他的cobra、plfag、viper... 为golang的生态完善做了很大贡献，致敬!

书归正传，hugo是基于golang开发的世界上最快的网站构建框架，本文介绍如何基于它构建技术类文档库，也可作为内部wiki或开源书籍使用。之所以没选择gitbook，发现CLI版本已于2018年停止维护了。

## 准备环境

安装git和hugo
 
- 具体过程此处不表。注意是安装hugo_extended的版本。
 
- 如果要编译成html，部署到webserver上，还需要安装nodejs 12+。
 

## 主题选择
选用Docsy主题，出自google的开源主题，很多流行项目使用此主题作为官方站点，有k8s、kubeflow、grpc、etcd、Selenium等，详见此。主要功能包含：
 - 支持树形目录
 - 国际化
 - 搜索功能
 - 移动端自适应适配
 - 标签分类
 - 全站打印
 - 文档版本化
 - 用户反馈等

官方提供了快速上手的脚手架，具体操作如下：
```bash
git clone https://github.com/google/docsy-example.git
``` 

脚手架初始化 
```bash
cd docsy-example && git submodule update --init --recursive
```

本地启动，默认可通过``localhost:1313``访问，官方提供了在线的[预览地址](https://example.docsy.dev/)。 
```bash
hugo server
```
​
## 目录结构说明

默认脚手架目录结构如下，只关注``config.toml``文件和``content``目录，就可以满足日常使用。
```bash
➜  docsy-example git:(master)  tree -L 1
.
├── assets # 静态资源
├── config.toml # 站点的配置文件：主题选择、名称、链接、页面分析、markdown解析引擎...
├── content # 站点内容：顶层导航，左侧树形章节
├── CONTRIBUTING.md
├── deploy.sh
├── docker-compose.yaml
├── Dockerfile
├── layouts # 可覆盖主题的默认布局，添加自定义页面布局
├── LICENSE
├── netlify.toml
├── package.json
├── README.md
├── resources
└── themes # Docsy主题的目录，hugo推荐的存放路径
```

## 重点说明

- 修改主题的页面布局
 
  - 大部分内容可以通过修改根目录的config.toml的文件来实现
  - 不能通过上条实现的，也不建议直接修改themes目录下的内容，copy到在根目录同样的相对路径上再修改，会覆盖默认主题相关的实现。

- docsy主题默认提供了3种版面布局, 分别在content目录下，写markdown文件，不同目录按照各自类型风格渲染。
 
  - docs技术文档类的布局
  - blog博客类的布局
  - community社区相关介绍的布局

- docs的目录章节可以任意分级，目录下的_index.md或者_index.html为该章、节的首页描述。
 
- 根目录运行如下命令可编译最终html，可能会提示``POSTCSS``编译失败，需要根目录运行``npm install``安装依赖。
  ```bash 
  rm -rf public/ && HUGO_ENV="production" hugo --gc || exit 1
  ```

- 站点内搜索功能依赖编译后的json文件，需要上一步才能使用。
 
- 关于config.toml配置的解读（最新注释说明，可查看本站的[原件](https://github.com/xiaoping378/xiaoping378.github.io/blob/master/config.toml)）。
  ```toml
  # 站点的访问地址，本地预览时(hugo server)可忽悠
  baseURL = "https://xiaoping378.github.io"

  # 站点title,会被多语言里的设置覆盖
  # title = "小平栈"

  # 是否生成robots文件
  enableRobotsTXT = true

  # 主题选择，支持组合，优先级从左到右.
  theme = ["docsy"]

  # 页面上提供类似"最后修改"的信息
  enableGitInfo = true

  # 国际化相关设置
  # 默认语言的的站点内容路径
  contentDir = "content"
  # 默认语言
  defaultContentLanguage = "zh-cn"

  # 国际化翻译中，如果有缺失是否用占位符显示
  enableMissingTranslationPlaceholders = true

  # 注释后，可以开启标签分类功能
  # disableKinds = ["taxonomy", "taxonomyTerm"]

  [params.taxonomy]
  # set taxonomyCloud = [] to hide taxonomy clouds
  taxonomyCloud = ["tags"] 
  # If used, must have same lang as taxonomyCloud
  taxonomyCloudTitle = ["标签"] 
  # set taxonomyPageHeader = [] to hide taxonomies on the page headers
  taxonomyPageHeader = ["tags"] 


  # 代码块高亮配置
  pygmentsCodeFences = true
  pygmentsUseClasses = false
  # Use the new Chroma Go highlighter in Hugo.
  pygmentsUseClassic = false
  #pygmentsOptions = "linenos=table"
  # See https://help.farbox.com/pygments.html
  pygmentsStyle = "emacs"

  # 配置blog编译产物的路径.
  [permalinks]
  blog = "/:section/:year/:month/:day/:slug/"

  # markdown渲染引擎配置: https://github.com/russross/blackfriday
  # [blackfriday]
  # plainIDAnchors = true
  # hrefTargetBlank = true
  # angledQuotes = false
  # latexDashes = true

  # 图片引擎处理: https://github.com/disintegration/imaging
  [imaging]
  resampleFilter = "CatmullRom"
  quality = 75
  anchor = "smart"

  # [services]
  # [services.googleAnalytics]
  # # Comment out the next line to disable GA tracking. Also disables the feature described in [params.ui.feedback].
  # id = "UA-00000000-0"

  # Language configuration

  [languages]
  [languages.zh-cn]
  title = "现代技能栈"
  description = "小平-所思所为"
  languageName = "中文"

  # 用于多语言排序，越小越靠上。
  weight = 1

  # markdown的解析设置，抄的k8s 文档设置...
  [markup]
    [markup.goldmark]
    [markup.goldmark.extensions]
      definitionList = true
      table = true
      typographer = false
    [markup.goldmark.parser]
      attribute = true
      autoHeadingID = true
      autoHeadingIDType = "blackfriday"
    [markup.goldmark.renderer]
      unsafe = true
    [markup.highlight]
      codeFences = true
      guessSyntax = false
      hl_Lines = ""
      lineNoStart = 1
      lineNos = false
      lineNumbersInTable = true
      noClasses = true
      style = "emacs"
      tabWidth = 4
    [markup.tableOfContents]
      endLevel = 3
      ordered = false
      startLevel = 2

  # Everything below this are Site Params

  # Comment out if you don't want the "print entire section" link enabled.
  [outputs]
  section = ["HTML", "print", "RSS"]

  [params]
  copyright = "xiaoping378"
  privacy_policy = "#"

  # First one is picked as the Twitter card image if not set on page.
  # images = ["images/project-illustration.png"]

  # Menu title if your navbar has a versions selector to access old versions of your site.
  # This menu appears only if you have at least one [params.versions] set.
  version_menu = "Releases"

  # Flag used in the "version-banner" partial to decide whether to display a 
  # banner on every page indicating that this is an archived version of the docs.
  # Set this flag to "true" if you want to display the banner.
  archived_version = false

  # The version number for the version of the docs represented in this doc set.
  # Used in the "version-banner" partial to display a version number for the 
  # current doc set.
  version = "0.0"

  # A link to latest version of the docs. Used in the "version-banner" partial to
  # point people to the main doc site.
  # url_latest_version = "https://example.com"

  # 方便用户反馈，提交技术文章问题的仓库地址
  github_repo = "https://github.com/xiaoping378/xiaoping378.github.io"
  # 技术站点背后的项目issue地址
  # github_project_repo = "https://github.com/xiaoping378/xiaoping378.github.io"

  # 以下三个是设置远程文档位置的，目前用不上，这里hack一下，不然“编辑此页”的功能会去链接到content/zh-cn下
  # Specify a value here if your content directory is not in your repo's root directory
  github_subdir = "/"

  # Uncomment this if you have a newer GitHub repo with "main" as the default branch,
  # or specify a new value if you want to reference another branch in your GitHub links
  # github_branch= "main"

  # 支持三种搜索，三选一，禁用google搜索，需要注释掉此处
  # gcs_engine_id = "d72aa9b2712488cc3"

  # Enable Algolia DocSearch
  algolia_docsearch = false

  # Enable Lunr.js offline search
  offlineSearch = false

  # 默认使用的Chroma代码高亮方案，可换成prism方案。
  prism_syntax_highlighting = false

  # User interface configuration
  [params.ui]
  #  是否禁用面包屑导航.
  breadcrumb_disable = false
  # 是否禁用底部About链接
  footer_about_disable = true
  # 是否展示项目logo，位置必须放置在 assets/icons/logo.svg
  navbar_logo = true
  # 在首页，上下滑动页面，顶部导航是否禁用半透明
  navbar_translucent_over_cover_disable = false
  # 左侧章节树形目录默认是否处于折叠状态
  sidebar_menu_compact = true
  # 左侧章节树形目录上是否不显示搜索框，前提是需要开启搜索功能
  sidebar_search_disable = false

  # 关闭了google分析，下面功能不会启用
  [params.ui.feedback]
  enable = true
  # The responses that the user sees after clicking "yes" (the page was helpful) or "no" (the page was not helpful).
  yes = 'Glad to hear it! Please <a href="https://github.com/USERNAME/REPOSITORY/issues/new">tell us how we can improve</a>.'
  no = 'Sorry to hear that. Please <a href="https://github.com/USERNAME/REPOSITORY/issues/new">tell us how we can improve</a>.'

  # 在文章上面显示“阅读时长：x分钟”
  [params.ui.readingtime]
  enable = false


  # 社区community版面要用到的参数
  [params.links]
  # End user relevant links. These will show up on left side of footer and in the community page if you have one.
  [[params.links.user]]
    name = "个人邮箱 xiaoping378@163.com"
    url = "mailto:xiaoping378@163.com"
    icon = "fa fa-envelope"
    desc = "欢迎邮件交流"
  [[params.links.user]]
    name ="微博"
    url = "https://weibo.com/xiaoping378"
    icon = "fab fa-weibo"
    desc = "个人微博，基本不用"
  [[params.links.user]]
    name = "知乎"
    url = "https://www.zhihu.com/people/xiaoping378"
    icon = "fab fa-zhihu"
    desc = "知乎专栏"
  # Developer relevant links. These will show up on right side of footer and in the community page if you have one.
  [[params.links.developer]]
    name = "GitHub"
    url = "https://github.com/xiaoping378/xiaoping378.github.io"
    icon = "fab fa-github"
    desc = "文集开源地址!"
  [[params.links.developer]]
    name = "Slack"
    url = "https://example.org/slack"
    icon = "fab fa-slack"
    desc = "未开通"
  [[params.links.developer]]
    name = "Developer mailing list"
    url = "https://example.org/mail"
    icon = "fa fa-envelope"
    desc = "未开通"
  ```

## 总结

不定期更新此文，待进一步优化，，，

如果自己搭建嫌麻烦，可直接Copy本站，编写自己的内容即可，搭建本站时，也遇到了Docsy国际化方面的问题，已提交官方[PR](https://github.com/google/docsy/pull/826)修复。

[下文](../actions_pages)介绍使用github的actions和pages自动托管站点。