# Conduit 概览
Conduit是一个针对Kubernetes的超轻量service mesh。它通过透明底管理服务间的运行时通信，使得在Kubernetes上运行服务更加安全和可靠；提供了针对可观测性、可靠性及安全性等方面的特性，而所有这些无需修改任何代码。

本文档将从一个较高层次介绍Conduit及其是如何工作的。如果不熟悉service mesh模型，或许可以先阅读William Morgan的概览文章 [什么是service mesh？为什么需要它？](https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/)

## Conduit架构
Conduit service mesh部署到Kubernetes集群时有两个基本组件：一个数据平面和一个控制平面。数据平面承载服务实例间实际的应用请求流量，而控制平面驱动数据平面并为修改其行为（及访问聚合指标）提供API。Conduit CLI和web UI使用这个API来为用户提供人体工学控制。

让我们依次认识这些组件。

Conduit的数据平面由轻量级的代理组成，这些代理作为sidecar容器与每个服务代码的实例部署在一起。要为Conduit servie mesh“增加”一个服务，该服务的pods必须重新部署，以便在每个pod中包含进一个数据平面。（`conduit inject` 命令负责这个，同时要通过代理透明地汇集每个示例的流量，配置工作也很必要。）

这些代理透明地拦截进出每个pod的通信，并增加诸如重试和超时、仪表及加密（TLS）等特性，甚至根据相关策略来允许和禁止请求。

这些代理并未设计成通过手动方式配置；相反，它们的行为是由控制平面驱动的。

Conduit控制平面是一系列服务，允许在一个专用的Kubernetes命名空间（默认是 `conduit`）。这些服务完成各种各样的任务——聚合遥测数据、提供一个面向用户的API、提供针对数据平面代理的控制数据，等等。总之，它们驱动数据平面的行为。

## 使用Conduit
为了支持Conduit的人机交互，可以使用Conduit CLI及web UI（也可以通过相关工具比如 `kubectl`）。CLI 和 web UI通过API驱动控制平面，而控制平面相应地驱动数据平面的行为。

控制平面API设计得足够通用，以便能基于此构建其他工具。比如，你可能希望另外从一个CI/CD系统来驱动API。

运行 `conduit --help` 可查看关于CLI功能的简短概述。

## Conduit 与 Kubernetes
Conduit设计用于无缝地融入现有的Kubernetes系统。该设计有几个重要特征。

第一，Conduit CLI（`conduit`）设计成尽可能地与 `kubectl` 一起使用。比如，`conduit install` 和 `conduit inject` 生成的Kubernetes配置，被设计成直接送入`kubectl`。这是为了在service mesh和orchestrator之间提供一个清晰的工作分工，且能使适配Conduit到已有Kubernetes工作流更加容易。

第二，Kubernetes中Conduit的核心词是Deployment，而不是Service。举个例子，`conduit inject` 增加一个Deployment，Conduit web UI会显示这些Deployments，并按Deployment提供聚合的性能指标。这是因为单个pods可以是任意数量Service的一部分，而这会导致通信流量与pods之间的复杂映射。相比之下，Deployment要求单个pod最多是一个Deployment的一部分。通过基于Deployments而不是Services来构建，流量与pod间的映射就总是清晰的。

这两个设计特性能很好地组合。比如，`conduit inject`可用于一个运行的Deployment，因为当它更新Deployment时， Kubernetes会回滚pods以包括数据平面代理。

## 扩展Conduit的行为
Conduit控制平面还为构建自定义功能提供了一个便捷的入口。Conduit初始发布时并不支持这一点，在不远的将来，通过编写作为控制平面的一部分运行的gRPC插件，将能扩展Conduit的功能，而无需重新编译Conduit。
