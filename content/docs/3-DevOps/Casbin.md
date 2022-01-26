---
title: "Casbin的权限管理解读"
linkTitle: "Casbin的权限管理解读"
weight: 8
description: >
  介绍Casbin的权限模型管理的用法。 
---

{{% pageinfo %}}
项目一般都要包含权限管理功能，或集成IAM，或自身实现。本文介绍一个强大、高效的开源访问控制框架--Casbin。
{{% /pageinfo %}}

## 基本介绍

Casbin的由来，是出自开源作者`罗杨`的一篇论文[《PML：一种基于Interpreter的Web服务访问控制策略语言》](https://arxiv.org/abs/1903.09756)，该论文的主要摘要如下：

> 为了保护云资源的安全,防止数据泄露和非授权访问,必须对云平台的资源访问实施访问控制.然而,目前主流云平台通常采用自己的安全策略语言和访问控制机制,从而造成两个问题:
> - （1）云用户若要使用多个云平台,则需要学习不同的策略语言,分别编写安全策略;
> - （2）云服务提供商需要自行设计符合自己平台的安全策略语言及访问控制机制,开发成本较高.
> 
> 对此,提出一种基于元模型的访问控制策略描述语言PML及其实施机制PML-EM. PML支持表达BLP、RBAC、ABAC等访问控制模型.
> PML-EM实现了3个性质:
> - 策略语言无关性
> - 访问控制模型无关性
> - 程序设计语言无关性
> 
> 从而降低了用户编写策略的成本与云服务提供商开发访问控制机制的成本. 在OpenStack云平台上实现了PML-EM机制.实验结果表明,PML策略支持从其他策略进行自动转换,
> 在表达云中多租户场景时具有优势.性能方面,与OpenStack原有策略相比,PML策略的评估开销为4.8%.PML-EM机制的侵入性较小,与云平台原有代码相比增加约0.42%. 


目前Casbin的权限策略管理支持主流的**ACL、RBAC、ABAC**、RESTful等模型，实现的编程语言主要有Go、java、Nodejs、PHP、Python、.Net、C++、Rust等。目前Go和Java的实现最为全面。

先介绍下主流的访问控制模型：
- _ACL（access control list）_：是一种与访问对象关联的权限列表，在基础设施领域应用非常广泛：
  - 文件系统：用户（组）对文件或进程等的访问权限控制
  - 网络：常见的有防火墙（安全组、路由器、交换机）内的对目的IP和端口的规则控制
  - SQL：库、表的权限管理
  - [LDAP](../openldap之拨云去日)：层级结构的实体权限管理：网络域权限管理...
- _RBAC（role-based access control）_：基于角色的权限控制，围绕角色和权限定义的策略中立的访问控制机制，和组ACL等价，具体表现为在用户和权限之间加了一层角色，先建立具有某种权限的角色，然后用角色和用户绑定，目前多用于业务系统内的权限管理，还支持支持角色权限继承。
- _ABAC（Attribute-based access control）_：基于属性的权限控制，属性可以是用户侧（所属组织、访问IP、访问时间）或资源侧（帖子的评论开关、留言再编辑）的，因为用户或资源的属性是动态的，不像前面两个(需要预先定义好策略，略显死板,,,）被称为是“下一代”的权限模型。
- _Restful_：一种Web API规范，常用GET, POST等动作来实现与后台交互。

和Casbin结合，使用的基本示意图如下：

![](/images/Casbin-使用示意图.drawio.png)

- 1-2为管理人员下发权限策略
- 3-6为用户日常操作资源的简易流程，实际应用场景一般如下：

```plantuml
@startuml
!theme aws-orange
用户 -> 认证中心: 登录操作
认证中心 -> 缓存: 存放(key=token+ip,value=token)token
用户 <- 认证中心 : 认证成功返回token
用户 -> 认证中心: 下次访问头部携带token认证
认证中心 <- 缓存: key=token+ip获取token
Casbin <- 认证中心: 存在且校验成功,则进入授权校验
Casbin -> 其他服务: 权限合规，则跳转到用户请求的其他服务
其他服务 -> 用户: 信息
@enduml
```


## 抽象模型

正如上面提到的，要支持这么多的权限模型，，所以Casbin基于开头提到的PML（PERM modeling language）引入一种抽象的元模型控制，其中**PERM**是指的**Policy, Effect, Request, Matchers**，具体工作流如下：

![](/images/Casbin-2022-01-26-23-21-26.png)

这里的`PERM`是Casbin在启动时要加载的抽象校验模型，可以理解成一种权限校验模板。 简单说个场景串下这里的概念：

- 管理员分配给用户权限**Policy**，可以得到`谁能操作什么资源`的信息
- 用户发起请求**Request**，可以得到`谁要操作什么资源`的信息
- Casbin拿到Request和Policy做匹配**Matcher**
- 根据匹配结果**Effect**来决定是否允许用户的操作

实际场景中，用户可能被分配了多个权限（角色），那具体权限校验如下：
![](/images/Casbin-2022-01-26-23-47-48.png)

## 按场景举例

TODO.

## 参考
> https://casbin.org/docs/zh-CN/tutorials

> 我的博客即将同步至腾讯云+社区，邀请大家一同入驻：https://cloud.tencent.com/developer/support-plan?invite_code=3ntkskjrcwow8