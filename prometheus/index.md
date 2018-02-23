# 使用Prometheus获取遥测数据

使用现有Prometheus集群获取Conduit的遥测数据是很容易的。简单的把下面的配置代码加入到Prometheus配置的`scrape_configs`之中即可：

~~~yaml
- job_name: 'conduit'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          # Replace this with the namespace that Conduit is running in
          names: ['conduit']
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        action: keep
        regex: ^admin-http$
~~~

这就可以了。现在你的Prometheus集群就已经配置完成，可以抓取Conduit的指标数据了。Conduit的指标会包含标签`job="conduit"`以及：

- `request_total`：请求总数
- `response_total`：响应总数
- `response_latency_ms`：毫秒为单位的响应延迟

所有的指标都带有下列标签：

- `source_deployment`：发出请求的Deployment（或者replicaset, job之类）
- `target_deployment`：接收请求的Deployment（或者replicaset, job之类）
