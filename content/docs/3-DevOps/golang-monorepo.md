---
tags: ["Dev"]
title: "Golang单仓monorepo协作的设计与实践"
linkTitle: "单仓协作管理"
weight: 10
description: >
 介绍Golang单仓monorepo协作的设计与实践
---

## 什么是monorepo

多代码仓合一就是nomorepo，目前在前端领域比较火热，国外大厂如Google、Facebook、Microsoft内也在公司级采用此模式，国内的浮躁，是个项目就上微服务架构的现状，导致代码仓动辄就上几十个，感觉也适合此模式。这东西说起来简单，实际运转起来，需要相应的支撑工具，如代码仓管理、CI/CD适配等，本文主要设计一套可行的方案及记录实践关键过程。

> 这里说下背景：为什么要研究这个，最近接手一个项目，发现团队积累几乎没有、需求响应进度慢、新成员接手难度大，另外该项目竟然有上百个仓库，哪些和线上正在运行的服务对应，也没有说明，，，相应的协作管理手段也薄弱，总之就是乱，。个人是信服`康威定律`（组织沟通方式会通过系统设计表达出来，反之亦成立）的，所以打算尝试下单仓的协作模式，一来可以梳理下以前的需求实现，二来促进组员交流沟通，制定些开发规范和统一框架之类的也好落地。

## 代码目录结构

## CODEOWNERS机制

## CI/CD机制

## 工具实践``Lighthouse``
