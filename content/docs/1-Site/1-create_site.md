---
tags: ["hugo", "Docsy"]
title: "站点搭建方法"
linkTitle: "站点搭建方法"
weight: 1
description: >
  主要介绍如何免费快速搭建自己的专属网络博客站点，hugo&Docsy主题
---

{{% pageinfo %}}
主要涉及到hugo和docsy主题的使用。
{{% /pageinfo %}}

## hugo工具介绍

hugo是spf13的开源作品，目前任职于google，对他最早的印象是使用他的vim-spf13配置，后来又接触到他的cobra、plfag、viper... 为golang的生态完善做了很大贡献，致敬!

书归正传，hugo是基于golang开发的世界上最快的网站构建框架，本文介绍如何基于它构建技术类文档库，也可作为内部wiki或开源书籍使用。之所以没选择gitbook，发现CLI版本已于2018年停止维护了。

## 准备环境

安装依赖工具（以下版本为当前202512最新稳定版本）：

- 安装git, 下载[地址](https://github.com/git-for-windows/git/releases/download/v2.52.0.windows.1/Git-2.52.0-64-bit.exe)
- 安装hugo_extended[Windows环境](https://github.com/gohugoio/hugo/releases/download/v0.152.2/hugo_extended_withdeploy_0.152.2_windows-amd64.zip)的版本。
- 安装Go工具，下载[地址](https://go.dev/dl/go1.24.10.windows-amd64.msi), hugo初始化需要go环境
- 安装nodejs 22+，下载[地址](https://nodejs.org/dist/v24.11.1/node-v24.11.1-win-x64.zip)，推荐使用fnm管理node多版本。

> 具体安装过程不表，下篇会介绍如何利用github pages全自动托管静态文件

## 主题选择
选用[Docsy主题](https://www.docsy.dev/xx/docs/get-started/)，出自google的开源主题，很多流行项目使用此主题作为官方站点，如k8s、kubeflow、grpc、etcd、Selenium等，详见此。主要功能包含：
 - 支持树形目录
 - 国际化
 - 搜索功能
 - 移动端自适应适配
 - 标签分类
 - 全站打印
 - 文档版本化
 - 用户反馈等
 - uml\mermaid渲染

官方提供了快速脚手架，具体操作如下：
```bash
git clone --depth 1 --branch v0.13.0 https://github.com/google/docsy-example.git my-new-site
cd  my-new-site
npm install --registry https://registry.npmmirror.com
hugo server
```

本地启动，默认可通过``localhost:1313``访问，官方提供了在线的[预览地址](https://example.docsy.dev/)。 
```bash
hugo server
```
​
## 目录结构说明

默认脚手架目录结构如下，只关注``hugo.yaml``文件和``content``目录，就可以满足日常使用。
```bash
➜  my-new-site git:(v0.13.0) ll
total 195K
drwxr-xr-x 1 xxp 197609    0 11月 27 20:34 assets # 静态资源目录，如CSS、JavaScript等
-rw-r--r-- 1 xxp 197609  448 11月 27 20:34 config.yaml  # 站点的hugo高版本兼容文件，可删除
drwxr-xr-x 1 xxp 197609    0 11月 27 20:34 content # 网站内容目录，包含Markdown格式的文章和页面，以后重点创作的目录
-rw-r--r-- 1 xxp 197609 1.1K 11月 27 20:34 CONTRIBUTING.md
-rw-r--r-- 1 xxp 197609  172 11月 27 20:34 docker-compose.yaml
-rw-r--r-- 1 xxp 197609  100 11月 27 20:34 Dockerfile
-rw-r--r-- 1 xxp 197609  173 11月 27 20:34 docsy.work
-rw-r--r-- 1 xxp 197609    0 11月 27 20:34 docsy.work.sum
-rw-r--r-- 1 xxp 197609   89 11月 27 20:34 go.mod
-rw-r--r-- 1 xxp 197609  513 11月 27 20:34 go.sum
-rw-r--r-- 1 xxp 197609 7.7K 11月 27 20:34 hugo.yaml #Hugo的主要配置文件
-rw-r--r-- 1 xxp 197609 7.5K 11月 27 20:34 hugo-disabled.toml
drwxr-xr-x 1 xxp 197609    0 11月 27 20:34 layouts
-rw-r--r-- 1 xxp 197609  12K 11月 27 20:34 LICENSE
-rw-r--r-- 1 xxp 197609  302 11月 27 20:34 netlify.toml # Netlify部署平台配置文件
drwxr-xr-x 1 xxp 197609    0 11月 27 20:37 node_modules
-rw-r--r-- 1 xxp 197609 2.7K 11月 27 20:34 package.json
-rw-r--r-- 1 xxp 197609  99K 11月 27 20:37 package-lock.json
drwxr-xr-x 1 xxp 197609    0 11月 27 20:37 public
-rw-r--r-- 1 xxp 197609 7.3K 11月 27 20:34 README.md
drwxr-xr-x 1 xxp 197609    0 11月 27 20:35 resources # 动态编译的产物
```

## 重点说明

- 修改主题的页面布局
 
  - 大部分渲染样式可以通过修改根目录的hugo.yaml的配置文件来实现
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
 
- 关于hugo.yaml配置的解读（最新注释说明，可查看本站的[原件](https://github.com/xiaoping378/xiaoping378.github.io/blob/master/hugo.yaml)）。
  ```yaml
  # 站点的访问地址，本地预览时(hugo server)可忽悠
  baseURL: "https://xiaoping378.github.io"

  # 站点title,会被多语言里的设置覆盖
  # title: "小平栈"

  # cSpell:ignore goldmark github hugo readingtime docsy subdir lastmod pygments linenos catmullrom norsk gu

  # Language settings
  contentDir: content
  defaultContentLanguage: zh-cn
  defaultContentLanguageInSubdir: false
  # 国际化翻译中，如果有缺失是否用占位符显示
  enableMissingTranslationPlaceholders: true

  # 是否生成robots文件
  enableRobotsTXT: true

  # 页面上提供类似"最后修改"的git信息
  enableGitInfo: true

  # 注释掉这行来禁用 Docsy 主题中的分类法功能
  # disableKinds: [taxonomy]

  # You can add your own taxonomies
  taxonomies:
    tag: tags
    category: categories

  # 代码块高亮配置
  pygmentsCodeFences: true
  # 开启后，黑色主题可以消除代码块的白色底色
  pygmentsUseClasses: false
  # Use the new Chroma Go highlighter in Hugo.
  pygmentsUseClassic: false
  # pygmentsOptions: "linenos=table"
  # See https://help.farbox.com/pygments.html
  pygmentsStyle: tango

  # 配置blog编译产物的路径.
  permalinks:
    blog: /:section/:year/:month/:day/:slug/

  # 图片处理的配置.
  imaging:
    resampleFilter: CatmullRom
    quality: 75
    anchor: Smart

  # 国际多语言配置
  languages:
    zh-cn:
      languageName: 中文
      title: 现代技能栈
      params:
        description: 现代技能栈、devops、云原生、网络、区块链、RPA、IOT、AI人工智能
    # en:
    #   languageName: English
    #   title: Goldydocs
    #   params:
    #     description: A Docsy example site
    # cSpell:disable


  # markdown的解析设置，抄的k8s的文档解析设置...
  markup:
    goldmark:
      extensions:
        definitionList: true
        table: true
        typographer: false
      parser:
        attribute:
          block: false
          title: true
        autoHeadingID: true
        autoHeadingIDType: "blackfriday"
      renderer:
        unsafe: true
    highlight:
      codeFences: true
      guessSyntax: false
      hl_inline: false
      lineAnchors: ''
      lineNoStart: 1
      lineNos: false
      lineNumbersInTable: true
      noClasses: true
      style: "emacs"
      tabWidth: 4
    tableOfContents:
      endLevel: 3
      ordered: false
      startLevel: 2

  # 此处以下均为站点参数。

  # 如果不需要启用“打印整个章节”链接，可以将其注释掉
  outputs:
    section: [HTML, print, RSS]

  params:
    # uml自动渲染绘图配置
    plantuml:
      enable: true
      theme: default
      # Set url to plantuml server
      # default is http://www.plantuml.com/plantuml/svg/
      svg_image_url: 'https://www.plantuml.com/plantuml/svg/'
      # By default the plantuml implementation uses <img /> tags to display UML diagrams.
      # When svg is set to true, diagrams are displayed using <svg /> tags, maintaining functionality like links e.g.
      # default = false
      svg: true
    # mermaid 绘图配置
    mermaid:
      theme: neutral
      flowchart:
        diagramPadding: 6

    taxonomy:
      # 如果不想显示分类标签云，可以将 taxonomyCloud: []设置为空数组。
      taxonomyCloud: [tags, categories]

      # 如果使用，其长度（即元素数量）必须与 taxonomyCloud相同。
      taxonomyCloudTitle: [标签, 分类]

      # 如果不想在页面标题（或页眉）处显示分类标签，可以将 taxonomyPageHeader设置为空数组 []
      taxonomyPageHeader: [tags, categories]

    privacy_policy: https://policies.google.com/privacy

    # First one is picked as the Twitter card image if not set on page.
    # images: [images/project-illustration.png]

    # Menu title if your navbar has a versions selector to access old versions of your site.
    # This menu appears only if you have at least one [params.versions] set.
    version_menu: Releases

    # Flag used in the "version-banner" partial to decide whether to display a
    # banner on every page indicating that this is an archived version of the docs.
    # Set this flag to "true" if you want to display the banner.
    archived_version: false

    # The version number for the version of the docs represented in this doc set.
    # Used in the "version-banner" partial to display a version number for the
    # current doc set.
    version: 0.0

    # A link to latest version of the docs. Used in the "version-banner" partial to
    # point people to the main doc site.
    url_latest_version: https://example.com

    # Repository configuration (URLs for in-page links to opening issues and suggesting changes)
    github_repo: https://github.com/xiaoping378/xiaoping378.github.io

    # 这是一个可选的相关项目仓库链接。例如，指向您产品代码所在的兄弟仓库（关联代码库）
    github_project_repo: https://github.com/google/docsy

    # Specify a value here if your content directory is not in your repo's root directory
    # github_subdir: ""

    # Uncomment this if your GitHub repo does not have "main" as the default branch,
    # or specify a new value if you want to reference another branch in your GitHub links
    github_branch: master

    # Google Custom Search Engine ID. Remove or comment out to disable search.
    gcs_engine_id: UA-217913492-1

    # Enable Lunr.js offline search
    offlineSearch: false

    # 使用 Prism 为代码块启用语法高亮和复制按钮功能
    prism_syntax_highlighting: true

    copyright:
      authors:
        作者 xiaoping378 | [CC BY 4.0](https://creativecommons.org/licenses/by/4.0) |
      from_year: 2018

    # User interface configuration
    ui:
      # 设置为 true 以禁用面包屑导航.
      breadcrumb_disable: false
      # 不想在顶部导航栏中显示logo（/assets/icons/logo.svg），请将此选项设置为 false。
      navbar_logo: true
      # 在页面滚动到 block/cover区块（例如首页的封面大图）上方时，顶部导航栏不呈现半透明效果，请将此选项设置为 true
      navbar_translucent_over_cover_disable: false
      # 让侧边栏菜单显示默认为折叠状态
      sidebar_menu_compact: true
      # 左侧导航显示 “可展开”的小三角
      sidebar_menu_foldable: true
      # 在网站中隐藏侧边栏的搜索框
      sidebar_search_disable: false
      # 导航栏中启用明暗模式切换菜单
      showLightDarkModeMenu: true

      # 在每个文档末尾添加一个标题为“反馈”的 H2 章节。用户反馈将作为事件发送到 Google Analytics（谷歌分析）
      # 此功能依赖于 [services.googleAnalytics] 配置，如果未设置 "services.googleAnalytics.id"，该功能将被禁用
      # 如果您启用了此功能，但需要偶尔在某个特定页面隐藏“反馈”章节,
      # 只需在该页面的 Front Matter 中添加 "hide_feedback: true" 即可.
      feedback:
        enable: true
        # 用户点击 “是”（表示此页有帮助）或 “否”（表示此页无帮助）后所看到的回复信息。
        'yes': >-
          与君同行!  <a
          href="https://github.com/xiaoping378/xiaoping378.github.io/issues/new">再接再厉</a>.
        'no': >-
          呃(⊙o⊙)…. 还请 <a
          href="https://github.com/xiaoping378/xiaoping378.github.io/issues/new">告知哪里可以改进</a>.

      # 在每个文档的顶部添加阅读时长估算.
      # 如果您启用了此功能，但需要偶尔在某个特定页面隐藏阅读时长
      # 只需在该页面的 Front Matter 中添加 "hide_readingtime: true" 即可
      readingtime:
        enable: false

    links:
      # End user relevant links. These will show up on left side of footer and in the community page if you have one.
      user:
        - name: 个人邮箱 xiaoping378@163.com
          url: mailto:xiaoping378@163.com
          icon: fa fa-envelope
          desc: 欢迎邮件交流
        - name: 微博
          url: https://weibo.com/xiaoping378
          icon: fab fa-x-twitter
          desc: 个人微博，基本不用
        - name: 知乎
          url: https://www.zhihu.com/people/xiaoping378
          icon: fab fa-stack-overflow
          desc: 知乎专栏
      # Developer relevant links. These will show up on right side of footer and in the community page if you have one.
      developer:
        - name: GitHub
          url: https://github.com/xiaoping378/xiaoping378.github.io
          icon: fab fa-github
          desc: 文集开源地址!
        - name: Gitee
          url: https://gitee.com/xiaoping378/xiaoping378
          icon: fa fa-git
          desc: 国内码云

  module:
    # Uncomment the next line to build and serve using local docsy clone declared in the named Hugo workspace:
    # workspace: docsy.work
    hugoVersion:
      extended: true
      min: 0.146.0
    imports:
      - path: github.com/google/docsy
        disable: false
  ```

## 总结

不定期更新此文，待进一步优化，，，

如果自己搭建嫌麻烦，可直接Copy本站，编写自己的内容即可，搭建本站时，也遇到了Docsy国际化方面的问题，已提交官方[PR](https://github.com/google/docsy/pull/826)修复。

[下文](../actions_pages)介绍使用github的actions和pages实现自动更新并托管站点。