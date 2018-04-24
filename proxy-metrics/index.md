# Proxy的监控指标

在流量通过Conduit代理的时候，会产生一系列用于表达流量特征的监控指标。下列的指标是Prometheus格式的，可以由Prometheus在Proxy监控端口（缺省为`4191`）的`/metrics`路径进行抓取。

> 下面每个指标的描述，会首先说明这个指标在 Prometheus 中的类型。

`request_total`

Counter，Proxy收到请求的数量。请求流启动的时候，这个数量会增加。

`request_duration_ms`

Histogram，用于描述请求的时长。从收到请求头开始计时，请求流完成后结束计时。

`response_total`

Counter，Proxy收到的响应的数量。当响应流结束时，这一数量会增加。

`response_duration_ms`

Histogram，描述响应的时长，从收到响应头开始计时，到响应流完成后结束计时。

`response_latency_ms`

Histogram，描述响应的总体延迟。这一时间段从收到请求头开始，到响应流完成结束。

## 标签

每个指标都会有如下的标签：

- `authority`：`:authority`（HTTP/2）或者`Host`（HTTP/1.1）请求头。
- `direction`：如果流量来自Pod之外，就是`inbound`；如果流量从Pod内发起，就是`outbound`。

### 响应标签

下列标签只会出现在`response_*`指标中：

- `classification`：如果响应成功，则取值`success`；如果服务器出错，则取值`failure`。如果存在gRPC状态码，则使用gRPC状态码决定取值，否则就使用HTTP状态码进行决定。
- `grpc_status_code`：来自`grpc-status`，仅对gRPC响应有效。
- `status_code`：响应的HTTP状态码。

### 出站标签

下列标签仅对`direction=outbound`有效：

- `dst_deployment`：请求发送的目标Deployment。
- `dst_k8s_job`：请求发向的Job。
- `dst_replica_set`：请求发送的目标RS。
- `dst_daemon_set`：请求发送的目标Daemonset。
- `dst_replication_controller`：请求发送的目标RC。
- `dst_namespace`：请求的目标所在命名空间。
- `dst_service`：请求发送的目标服务。

### Prometheus 收集器标签

Prometheus 收集器会在指标上加入如下标签：

- `instance`: Pod的ip:port。
- `job`：Prometheus Job的名称，通常是`conduit-proxy`。

### 收集时加入的 Kubernetes 标签

Kubernetes命名空间、Pod名称以及所有标签都会映射为Prometheus标签。

- `namespace`：Pod 所属的 Kubernetes 命名空间。
- `pod`：Kubernetes Pod 名称。
- `pod_template_hash`：对应到Kubernetes之中[`pod-template-hash`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#pod-template-hash-label)。这个值在重新部署或者滚动重启的时候会发生变化。

### 收集时加入的Conduit标签

- 在使用`conduit inject`进行注入时，会将`conduit.io/`为前缀的标签加入应用。确切的说，`conduit.io/proxy-*`前缀的Kubernetes标签对应下面的Prometheus标签，这些标签是否存在要视乎Pod所属的控制器：

- `daemon_set`：Pod所属的Daemonset。
- `deployment`：Pod所属的Deployment。
- `k8s_job`：Pod所属的Job。
- `replica_set`：Pod所属的RS。
- `replication_controller`：Pod所属的RC。

## 示例

这里是一个Pod的代码片段：

~~~yaml
name: vote-bot-5b7f5657f6-xbjjw
namespace: emojivoto
labels:
  app: vote-bot
  conduit.io/control-plane-ns: conduit
  conduit.io/proxy-deployment: vote-bot
  pod-template-hash: "3957278789"
  test: vote-bot-test
~~~

会生成的Promethus标签：

~~~console
request_total{
  pod="vote-bot-5b7f5657f6-xbjjw",
  namespace="emojivoto",
  app="vote-bot",
  conduit_io_control_plane_ns="conduit",
  deployment="vote-bot",
  pod_template_hash="3957278789",
  test="vote-bot-test",
  instance="10.1.3.93:4191",
  job="conduit-proxy"
}
~~~