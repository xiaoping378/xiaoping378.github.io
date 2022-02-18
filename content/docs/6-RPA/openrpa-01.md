---
tags: ["rpa"]
title: "openrpa介绍"
linkTitle: "openrpa介绍"
weight: 1
description: >
  开源openrpa的架构介绍
---

{{% pageinfo %}}
openrpa的架构介绍
{{% /pageinfo %}}

## 概述

openrpa项目，提供了**OON**技术栈（OpenRPA、Openflow、Node-RED）来实现的RPA功能。

**OpenRPA**

该组件可以认为是机器人的IDE设计器部分，基于[MWF](https://docs.microsoft.com/en-us/dotnet/framework/windows-workflow-foundation/)框架实现的，
- 可以图形化拖拽的方式实现基本功能`Activity`，如点击邮箱、打开浏览器网址、输入文字、录屏录音、各类监听探测器等等
- 多个`Activity`可以组合成一个序列，来实现工作流`workflow`，串行执行一系列动作
- 也可以通过配置作为agent（机器人）运行，负责执行设计好的工作流
- 默认远程连接到云端控制端（openflow），可上传同步本地的工作流和接受控制端的调度管理。

![](/images/openrpa-01-2022-02-14-13-54-44.png)

**Openflow**

OON技术栈的后台核心部分，可以关编排管理所有的活动，主要具备一下特性：
- 管理，调用和配置机器人和工作流
- 管理用户和权限(组)
- 创建Web表单，供和真实用户交互，实现工作流的输入，审批和确认等操作。
- 管理后端MongoDB存储
- 存储工作流、表单等数据信息

![openflow with Monitoring](/images/openrpa-01-2022-02-15-09-23-27.png)

**Node-RED**


## FAQ

三大组件汇总具有`流`的概念，openrpa中叫`workflow`，node-red中叫`flow`, openflow组件名称自身就带了flow的字样，界面上也有