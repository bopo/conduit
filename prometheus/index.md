# 使用Prometheus获取遥测数据

使用现有Prometheus集群获取Conduit的遥测数据是很容易的。简单的把下面的配置代码加入到Prometheus配置的`scrape_configs`之中即可：

~~~yaml
- job_name: 'conduit-controller'
      kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['{{.Namespace}}']
      relabel_configs:
      - source_labels:
        - __meta_kubernetes_pod_label_conduit_io_control_plane_component
        - __meta_kubernetes_pod_container_port_name
        action: keep
        regex: (.*);admin-http$
      - source_labels: [__meta_kubernetes_pod_container_name]
        action: replace
        target_label: component

    - job_name: 'conduit-proxy'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels:
        - __meta_kubernetes_pod_container_name
        - __meta_kubernetes_pod_container_port_name
        action: keep
        regex: ^conduit-proxy;conduit-metrics$
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: pod

      # 注意这里给k8s job设置的标签，不要和Prometheus的"Job"混淆在一起。
      # __meta_kubernetes_pod_label_conduit_io_proxy_job=foo =>
      # k8s_job=foo
      - source_labels: [__meta_kubernetes_pod_label_conduit_io_proxy_job]
        action: replace
        target_label: k8s_job
      # __meta_kubernetes_pod_label_conduit_io_proxy_deployment=foo =>
      # deployment=foo
      - action: labelmap
        regex: __meta_kubernetes_pod_label_conduit_io_proxy_(.+)
      # 删掉所有之前做了映射的标签。
      - action: labeldrop
        regex: __meta_kubernetes_pod_label_conduit_io_proxy_(.+)
      # __meta_kubernetes_pod_label_foo=bar => foo=bar
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
~~~

这就可以了。现在你的Prometheus集群就已经配置完成，可以抓取Conduit的指标数据了。

Conduit的Proxy指标会带有标签`job="conduit-proxy"`。Conduit的控制平面指标的标签是`job="conduit-controller"`。

要了解更多特定指标和标签的定义，请阅读[Proxy的监控指标](../proxy-metrics)。