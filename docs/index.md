# Conduit概览

Conduit是一个面向Kubernetes的超轻量Service Mesh。他对运行在Kubernetes中的服务间通信进行透明的管理，让服务变得更加安全和可靠。在不需要变更代码的前提下，Conduit微服务提供了可靠性、安全性和可监控的特性。

本文档将高层次的概述Conduit，及其它是如何工作的。如果不熟悉service mesh模型，可以先阅读William Morgan的概览文章[What’s a service mesh? And why do I need one?](https://buoyant.io/2017/04/25/whats-a-service-mesh-and-why-do-i-need-one/)

> 译者注：
> 1. [什么是服务网格以及为什么我们需要服务网格？](http://www.infoq.com/cn/news/2017/11/WHAT-SERVICE-MESH-WHY-NEED) 这是上文的中文翻译版本
>
> 2. [Awesome Service Mesh资料清单](https://github.com/ServicemeshCN/awesome-servicemesh): 可以从这里得到更多的Service Mesh资料，持续更新中

## Conduit架构

Conduit service mesh部署到Kubernetes集群时有两个基本组件：_数据平面_和_控制平面_。数据平面承载服务实例间的实际应用请求流量，而控制平面驱动数据平面，并提供API以修改其行为（还有访问聚合指标）。Conduit CLI和web UI使用这个API，并提供适用于人类的人体工学控制。

让我们依次认识这些组件。

Conduit的数据平面由轻量级的代理组成，这些代理部署为sidecar容器，与每个服务代码的实例在一起。要“增加”服务到Conduit servie mesh，该服务的pods必须重新部署，以便在每个pod中包含一个数据平面。（`conduit inject` 可以完成这个任务，以及必要的配置工作，以便通过代理来从每个实例透明的获取流量。）

这些代理透明地拦截进出每个pod的通信，并增加诸如重试和超时、仪表及加密（TLS）等特性，甚至根据相关策略来允许和禁止请求。

这些代理并未设计成通过手动方式配置；相反，它们的行为是由控制平面驱动的。

Conduit控制平面是一系列服务，运行在专用的Kubernetes命名空间（默认是 `conduit`）。这些服务完成各种的任务——聚合遥测数据、提供面向用户的API、为数据平面代理提供控制数据，等等。总之，它们驱动数据平面的行为。

## 使用Conduit

为了支持Conduit的人机交互，可以使用Conduit CLI及web UI（也可以通过相关工具比如 `kubectl`）。CLI和web UI通过API驱动控制平面，而控制平面相应地驱动数据平面的行为。

控制平面API设计的足够通用，以便能基于它构建其他工具。比如，你可能希望另外从一个CI/CD系统来驱动API。

运行 `conduit --help` 可查看关于CLI功能的简短概述。

## Conduit与Kubernetes

Conduit设计为可以无缝地融入现有的Kubernetes系统。该设计有几个重要特征。

首先，Conduit CLI（`conduit`）设计成尽可能与 `kubectl` 一起使用。比如，`conduit install` 和 `conduit inject` 生成的Kubernetes配置，被设计成可以直接送入`kubectl`。这是为了在service mesh和orchestrator之间提供一个清晰的工作分工，且能更加容易地适配Conduit到已有Kubernetes工作流。

其次，Conduit在Kubernetes中的核心名词是Deployment，而不是Service。举个例子，`conduit inject` 增加Deployment，Conduit web UI显示Deployments，并按Deployment提供聚合的性能指标。这是因为单个pods可以是任意数量Service的一部分，而这会导致通信流量与pods之间的复杂映射。相比之下，Deployment要求单个pod最多是一个Deployment的一部分。通过基于Deployment而不是Service来构建，流量与pod间的映射就总是清晰的。

这两个设计特性可以良好组合。比如，`conduit inject`可用于一个运行的Deployment，因为当它更新Deployment时， Kubernetes会回滚pods以包含数据平面代理。

## 扩展Conduit的行为

Conduit控制平面还为构建自定义功能提供便利。Conduit初始发布时并不支持这一点，在不远的将来，可以通过编写gRPC插件，作为控制平面的一部分运行，来扩展Conduit的功能，而无需重新编译Conduit。
