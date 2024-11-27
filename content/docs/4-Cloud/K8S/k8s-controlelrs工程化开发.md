---
tags: ["容器", "K8S", "controller"]
title: "k8s controllers工程化实践"
linkTitle: "controllers工程化实践"
weight: 21
description: >
  k8s controllers工程化实践总结。 
---

# controllers工程化

## 工程化的目标

controller工程化的定义，建立一个可持续迭代的工程，包括但不限于以下目标。

- 支持多group资源（多个controller）
- 更改CR字段后，可无缝升级（重新生产CR和API）
- API文档化
- CR部署初始化
- ARM多架构编译和镜像构建
- 单元测试覆盖率，golang-ci代码扫描。
- 暴露关键的监控指标和事件日志
- 高可用
- 关注规模性能
- 安全问题
  - webhook证书，统一管理
  - 组件Token权限
  - CR幂等性


## 创建一个Operator

利用kubebuilder初始化一个Operator，背后依赖controller-runtime和controller-gen

```bash
mkdir -p ~/app && cd ~/app

kubebuilder init --domain cebpaas.io --repo cebpaas.io/appmanager
# Writing kustomize manifests for you to edit...
# Writing scaffold for you to edit...
# Get controller runtime:
# $ go get sigs.k8s.io/controller-runtime@v0.11.2
# Update dependencies:
# $ go mod tidy
# Next: define a resource with:
# $ kubebuilder create api

#可选命令, 本处执行的话，可省略下文的“多个controller合并”
#kubebuilder edit --multigroup=true

kubebuilder create api --group apps --version v1 --kind Application
# Create Resource [y/n]
# y
# Create Controller [y/n]
# y
# Writing kustomize manifests for you to edit...
# Writing scaffold for you to edit...
# api/v1/application_types.go
# controllers/application_controller.go
# Update dependencies:
# $ go mod tidy
# Running make:
# $ make generate
# mkdir -p /root/app/bin
# GOBIN=/root/app/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.8.0
# /root/app/bin/controller-gen object:headerFile="hack/boilerplate.go.txt" paths="./..."
# Next: implement your new API and generate the manifests (e.g. CRDs,CRs) with:
# $ make manifests
make manifests
# /root/app/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
```

## 多个group controller合并

开启多controller操作，

```bash
# 开启多controller
kubebuilder edit --multigroup=true
# 手动修复之前默认创建的单仓，apps是之前示例中创建的group
mkdir apis/apps
mv api/* apis/apps
# After ensuring that all was moved successfully remove the old directory `api/`
rm -rf api/ 

mkdir controllers/apps
mv controllers/* controllers/apps/

# 修改之前go文件的import错误和package名称（controllers->apps）

# 修改`controllers/<group>/suite_test.go`文件中的CRDDirectoryPaths路径错误
# CRDDirectoryPaths: []string{filepath.Join("..", "..", "config", "crd", "bases")},
```

如上修改完毕，就可以添加新的group资源了.

```bash
kubebuilder create api --group cronhpacontroller  --version v1 --kind Cronhpa
```

最终的目录结构
```bash
tree -L 2  
.
├── apis
│   ├── apps # 同group多个kind资源，会默认生成在此目录
│   └── cronhpacontroller # 此处为新添加的group
├── bin
│   └── controller-gen
├── config
│   ├── crd
│   ├── default
│   ├── manager
│   ├── prometheus
│   ├── rbac
│   └── samples
├── controllers
│   ├── apps
│   └── cronhpacontroller
├── Dockerfile
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
├── main.go
├── Makefile
├── PROJECT
└── README.md
```

## 小计
- **controller-runtime架构**
![](/static/images/controller工程化-share-2023-03-14-23-32-21.png)

- **controller-gen**, 根据go文件里的标记注释，按规则自动生成DeepCopy代码、CR manifest、Webhook、Role等对象。
  - `bin/controller-gen -hh` 查看命令可用参数，内置5类生成器。
  - `bin/controller-gen crd -ww` 查看各类生成器支持的标记注释。

- **code-generator**
  - KCP使用了该工具，该工具集成在k8s主仓库中，内置多种生成器(deepcopy、informers、listers、**clientsets**、**openapi**等)，相对更底层，需要自己封装实现controller-runtime的功能，官方给出了可参考的示例`sample-controller`。
  - ClientSet提供了如k8s内置资源的便捷操作方法，可避免使用DynamicClient去操作非结构化数据结构。
  - client-go支持RESTClient、ClientSet、DynamicClient、DiscoveryClient四种客户端。

- Operator开发有多种方案
  - `Kubebuilder`(controller-runtime + controller-gen)
  - `Code-generator` + sample-controller
  - `Operator SDK` 基于kubebuidler扩展了更多的企业级功能，如OLM、OperatorHub和其他技术栈（ansible、helm）的Operator能力。
  - 其他

- 目前ACP中，前两种都用到了，推测主要原因是kubebuilder v2版本不支持mutli-group特性。

- kubebuilder自动创建api时，可以选择是否生产controller，如果只选择生成resource，相当于只创建CR注册和安装初始化的内容（生成apis目录下的 _types.go和deepcopy代码）。