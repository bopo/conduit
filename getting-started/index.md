# 入门

Conduit是一个面向Kubernetes的超轻量Service Mesh。他对运行在Kubernetes中的服务间通信进行透明的管理，让服务变得更加安全和可靠。在不需要变更代码的前提下，Conduit微服务提供了可靠性、安全性和可监控的特性。

Conduit有两个基础组件：_数据平面_ 是一个轻量级代理，他会以sidecar容器的形式和服务并行；_控制平面_ 则用来对这些代理进行协调和管理。用户可以使用命令行界面（CLI）来和Service Mesh进行交互，另外还有Web应用用于控制集群。

本文会谈到如何在Kubernetes集群中部署Conduit，并在其上运行一个gRPC应用。注意Conduit 0.1是一个Alpha版本。他Alpha到什么程度呢？目前仅提供了HTTP/2（包括gRPC），甚至HTTP/1.1也不被支持。如果你没有HTTP/2应用也不用担心，我们已经提供了例子进行试验。

## 先决条件

需要运行Kubernetes 1.8版本的集群，本机可正常工作的`kubectl`命令。可以用下面的命令来检查是否符合需要：

~~~
$ kubectl version --short
~~~

应该会看到这样的返回内容：

~~~
Client Version: v1.8.3
Server Version: v1.8.0
~~~

请确认"Server Version"内容是1.8或以上。如果返回内容不是上面的情况，或者`kubectl`返回了错误信息，可能是集群故障或者没有正确配置（如果想要快速简便的在本机运行Kubernetes，我们建议使用[Minikube](https://kubernetes.io/docs/getting-started-guides/minikube/)，需要注意的是，要求0.24.1或其后的版本。）

## 安装CLI

如果你是首次运行Conduit，需要下载命令行界面工具（CLI）到本机，然后使用CLI在Kubernetes集群上安装Conduit。

运行下列命令，开始安装CLI：

~~~
curl https://run.conduit.io/install | sh
~~~

跟随其中的指示进行操作即可。

另外，还可以直接在[Conduit发布页面](https://github.com/runconduit/conduit/releases)下载CLI；或者可以从[Conduit Github](https://github.com/runconduit/conduit)下载源代码，根据README指导自行Build。

CLI安装之后，用下面命令检查一下运行状况：

~~~
conduit version
~~~

应该会看到类似这样的内容：

~~~
Client version: v0.1.0
Server version: unavailable
~~~

## 在集群上安装Conduit

安装好了CLI之后，就可以在Kubernetes集群上安装Conduit的控制平面了。不要担心目前集群上运行的其他应用——控制平面会安装在单独的`conduit`命名空间，所以要删除也是很容易的。

运行下面的命令：

~~~
conduit install | kubectl apply -f -
conduit dashboard
~~~

第一个命令生成了Kubernetes配置，然后用管道输出给`kubectl`，Kubectl会把配置发送到Kubernetes集群。（为了观察细节，可以把这个命令拆开，例如`conduit install > foo.yml; cat foo.yml`）。

第二个命令在浏览器窗口中加载了一个仪表盘，如果你看到下图类似的内容，那么恭喜你，Conduit已经在你的集群上运行了。

![Conduit Dashboard](images/dashboard.png)

当然，目前还没有任何服务运行在Service Mesh之中，所以这个仪表盘很空旷，只显示了Service Mesh自身的情况。

## 安装演示应用

终于来到演示应用的安装环节了。用下面的命令可以运行应用：

~~~
curl https://raw.githubusercontent.com/runconduit/conduit-examples/master/emojivoto/emojivoto.yml | conduit inject - --skip-inbound-ports=80 | kubectl apply -f -
~~~

上面的命令下载了一个示例gRPC应用的Kuberneetes配置，然后用`conduit inject`运行。诸如操作把Conduit的数据平面的代理服务作为sidecar容器插入到应用Pod中。最后由`kubectl`把这些配置发送给Kubernetes集群之中。

跟`conduit install`类似，Conduit CLI做了些文本处理，然后由`kubectl`把配置提交给Kubernetes集群。可以再管道中加入其它的过滤器，或者分开运行来观察中间步骤的结果。

至此，这个应用就在Kubernetes集群上运行了，并且也被加入到了Conduit的Service Mesh之中。

## 观察运行情况

浏览一下Conduit的仪表盘，会看到加入Conduit Mesh中的用用列表中，我们的演示应用的所有HTTP/2通信（以后的版本中会显示其他协议的服务）。

我们可以用下面的URL查看一下这个应用：

~~~bash
minikube -n emojivoto service web-svc --url # on minikube
kubectl get svc web-svc -n emojivoto -o jsonpath="{.status.loadBalancer.ingress[0].*}" # on GKE
~~~

为了让仪表盘生动一些，我们可以编程生成一些流量，首先我们需要IP地址：

~~~bash
INGRESS=$(minikube -n emojivoto service web-svc --url | cut -f 3 -d / -) # on minikube
INGRESS=$(kubectl get svc web-svc -n emojivoto -o jsonpath="{.status.loadBalancer.ingress[0].*}") # on GKE
~~~

现在我们用一个简单的循环来生成流量：

~~~bash
while true; do curl $INGRESS; done
~~~

最后，我们再看仪表盘（如果尚未运行，执行`conduit dashboard`）。会看到应用中所有的服务：

- 成功率
- 请求频率
- 延迟的分布区间
- 上下游依赖

还有一些其他的流量信息。

## 使用CLI

当然了，仪表盘不是观测Conduit Service Mesh的唯一手段。CLI提供了很多有趣又强大的命令，包括：

- `conduit stat`
- `conduit tap`

试试看运行`conduit stat deployments`和`conduit tap deploy default/xxx`，看看会有什么结果。

## 总结

上面就是一个入门的简介。可以查看[综述](https://conduit.io/docs)和[路线图](https://conduit.io/roadmap)，或者加入[Linkerd Slack](https://slack.linkerd.io/)的`#conduit`频道，以及浏览[Conduit论坛](https://discourse.linkerd.io/c/conduit)。还可以在Twitter上关注[@runconduit](https://twitter.com/runconduit)。我们刚刚开始Conduit的工作，非常需要用户的反馈。
