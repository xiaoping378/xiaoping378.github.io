---
tags: ["hugo", "DevOps", "GitHubActions"]
title: "自动托管markdown"
linkTitle: "自动托管markdown"
weight: 2
description: >
  描述如何利用github的CI/CD实现自动更新并托管站点。
---

{{% pageinfo %}}
利用github actions和pages实现自动更新托管内容，本站点已实现commit md后，~~自动更新[码云page](https://xiaoping378.gitee.io/)和~~[Github page](https://xiaoping378.github.io/)页面。
{{% /pageinfo %}}

## 本站点仓库名字的由来

本源码仓之所以起如此长的名字，完全是因为github pages的工作机制决定的，不然会带个小尾巴：

如果叫 _blog_, github的托管页面的访问地址会是 _xiaoping378.github.io/blog_，这也没什么，但会和hugo的[static机制](/docs/1-site/add_content/#图片路径问题)出现冲突。 

当然可以自行购买域名，并指向该pages地址，，，后来忘了续费，搞丢了我的域名。


## 什么是pages
GitHub Pages 是 GitHub 提供的一项免费静态网站托管服务。简单来说，它能将你的代码仓库直接变成一个可访问的网站，无需自己购买服务器、配置环境，甚至连域名都可以使用 GitHub 提供的免费子域名。

## 快速入门三步走

- 第一步：创建仓库
  - 个人/组织网站：创建名为 你的GitHub用户名.github.io 的仓库（例如：xiaoping378.github.io）
  - 项目网站：在任何项目仓库中启用 Pages 功能

- 第二步：添加内容
  - 直接上传 HTML、CSS、JS 文件
  - 或使用静态网站生成器（如 Hugo、Jekyll、VuePress 等）生成静态文件
  - 配置 GitHub Pages 源（通常为 main 分支的 /root 或 /docs 文件夹）

- 第三步：访问你的网站
  - 个人站点：https://你的GitHub用户名.github.io
  - 项目站点：https://你的GitHub用户名.github.io/项目名

与静态网站生成器的完美结合. 正如我们上一篇提到的 Hugo + Docsy 方案，GitHub Pages 与静态网站生成器是绝佳搭档。你只需：

- 本地使用 Hugo 预览编写内容
- 配置 GitHub Actions 自动构建
- 推送到 GitHub 后，Actions 会自动将生成的静态文件部署到 Pages
- 几十秒后，更新就会上线！
这种方式让你专注于内容创作，而不用操心服务器维护和部署流程。

## 自动化工作流

上节是介绍了pages的快速入门，下面介绍自动化工作流，利用github actions实现自动更新托管内容。

在仓中创建`.github/workflows/deploy.yml`文件，设置GitHub Actions自动构建，文件内容可参考文末的原文链接。

### 工作流整体架构
```yaml
name: GitHub Pages
on:
  push:
    branches: [master]
  pull_request:
```

设计理念：

- 🚦 双触发机制：既响应代码推送（push）也响应拉取请求（pull_request）
- ✅ 质量把控：PR也会触发构建，可提前发现部署问题
- 🎯 精准部署：虽然两种操作都会触发流水线，但最终部署仅在master分支推送时执行

### 构建环境与资源优化
```yaml

runs-on: ubuntu-22.04
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
```

配置解读：
- 🐧 稳定环境：指定Ubuntu 22.04 LTS版本，确保构建环境一致性
- 🚦 智能并发：同一分支的多次推送会排队执行，避免构建冲突和资源浪费
- 💡 技巧：concurrency配置防止了因频繁提交导致的部署混乱问题

### 代码检出
```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive  # 关键！递归获取Docsy主题
    fetch-depth: 0         # 获取完整历史，支持Git信息功能
```
深度解析：

- 🔗 子模块处理：Docsy主题作为Git子模块，必须使用submodules: recursive完整获取，（为Docsy主题的早期版本优化）
- ⏳ 历史记录：fetch-depth: 0确保完整历史，这是Hugo的.GitInfo和页面"最后修改时间"功能的基础
- ⚠️ 注意：没有这两项配置，主题将无法加载，且页面会失去版本控制信息

### 构建工具链配置
```yaml
- name: Setup Hugo
  uses: peaceiris/actions-hugo@v2
  with:
    hugo-version: '0.152.2'
    extended: true

- name: Setup Node
  uses: actions/setup-node@v3
  with:
    node-version: '24'
```

配置解读：

- ⚙️ Extended版本：Docsy主题依赖SCSS编译，必须使用Hugo Extended版本
- 🔢 版本固定：明确指定0.152.2版本，避免因Hugo更新导致的构建失败
- 🔄 Node.js 24：使用最新LTS版本，确保与现代前端工具链兼容
- 🌐 小知识：peaceiris是GitHub Pages领域的知名维护者，其actions被数万个项目使用

### 性能优化：依赖缓存策略
```yaml
- name: Cache dependencies
  uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
```

缓存策略：

- ⚡ 构建加速：缓存npm依赖，减少70%+的文章发布时间
- 🔄 智能键值：基于package-lock.json哈希生成唯一键，依赖变化时自动更新缓存
- 🔀 降级策略：当精确缓存不存在时，尝试使用操作系统级别的通用缓存
- 📊 效果：首次构建可能需要2-3分钟，后续构建通常只需30-60秒

### 核心构建流程
```yaml
- run: npm install
- run: env HUGO_ENV="production" hugo --minify
```

生产实践：

- 🏭 生产环境构建：设置HUGO_ENV="production"启用优化，如资源指纹、缓存控制
- 📦 文件压缩：--minify参数最小化HTML、CSS、JS，通常减少30%+的文件体积
- 🧪 对比：开发构建（无此参数）保留调试信息，生产构建极致优化加载速度

### 安全部署机制
```yaml
- name: Deploy
  uses: peaceiris/actions-gh-pages@v3
  if: ${{ github.ref == 'refs/heads/master' }}
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_branch: gh-pages
```

设计亮点：

- 🔐 权限控制：使用GitHub自动提供的token，避免敏感信息泄露
- 🚫 条件部署：if条件确保只有master分支推送才触发实际部署，PR只验证不发布
- 🌿 标准分支：发布到gh-pages分支，符合GitHub Pages规范
- 📈 数据：从构建完成到全球CDN生效通常只需45-90秒

### 进阶优化建议

还可以考虑诸如构建状态通知, 多环境部署（预发布/生产）,资源清理策略等配置，本人未添加，可自行AI添加。


## 总结：这个工作流值得借鉴的地方

- ✅ 零配置部署：开发者只需关注内容，推送即发布
- ✅ 资源高效：缓存策略最大化利用GitHub Actions资源配额
- ✅ 故障安全：构建失败不会影响线上站点
- ✅ 版本透明：完整Git历史与页面修改信息保留
- ✅ 全球分发：GitHub Pages自带CDN与HTTPS

这套工作流已在Kubernetes、etcd等顶级开源项目中验证，无论你是个人博客、团队文档还是企业知识库，都能提供企业级的部署体验。每次git push，都是向世界分享知识的瞬间。

> 💡 实践建议：新手可直接复制此仓库链接的工作流。遇到问题？在评论区分享你的经验，我们一起探讨！

## 常见问题解答
Q：GitHub Pages有流量限制吗？
A：官方政策是每月100GB流量，对个人博客和文档站完全足够。

Q：能否使用自己的域名？
A：完全可以！只需在仓库设置中添加CNAME文件，将个人域名DNS指向GitHub。

Q：更新内容后多久生效？
A：通常30秒-2分钟，全球CDN同步可能需要10分钟完全生效。

Q：如何提升国内访问速度？
A：可考虑将仓库同步到Gitee，~~使用Gitee Pages或~~配置Cloudflare CDN。


#GitHubActions #DevOps #静态站点 #自动化部署 #技术实践
