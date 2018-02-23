# 示例：给应用除错

这里首先假设读者已经跟随[入门指南](../getting-started)进行了安装配置，Conduit和演示应用都已经在Kubernetes集群上运行。

## 用Conduit为故障服务排错

Conduit及其演示应用已经[启动运行](../getting-started)，我们接下来使用Conduit来检测问题。

首先，使用`conduit stat`命令来获取部署健康情况：

```
conduit stat deployments
```

大概会看到类似下面的输出内容：

```
NAME                   REQUEST_RATE   SUCCESS_RATE   P50_LATENCY   P99_LATENCY
emojivoto/emoji              2.0rps        100.00%           0ms           0ms
emojivoto/voting             0.6rps         66.67%           0ms           0ms
emojivoto/web                2.0rps         95.00%           0ms           0ms
```

我们会发现`voting`服务比其他情况差了很多。

我们是如何完成这一发现的？过去的办法是：看日志、加入调试器等。Conduit提供了新的工具，可以查看部署内的流量情况。我们可以使用`tap`命令来查看当前请求在部署中的流动。

```
conduit tap deploy emojivoto/voting
```

会出现大量的请求：

```
req id=0:458 src=172.17.0.9:45244 dst=172.17.0.8:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VoteGhost
rsp id=0:458 src=172.17.0.9:45244 dst=172.17.0.8:8080 :status=200 latency=758µs
end id=0:458 src=172.17.0.9:45244 dst=172.17.0.8:8080 grpc-status=OK duration=9µs response-length=5B
req id=0:459 src=172.17.0.9:45244 dst=172.17.0.8:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VoteDoughnut
rsp id=0:459 src=172.17.0.9:45244 dst=172.17.0.8:8080 :status=200 latency=987µs
end id=0:459 src=172.17.0.9:45244 dst=172.17.0.8:8080 grpc-status=OK duration=9µs response-length=5B
req id=0:460 src=172.17.0.9:45244 dst=172.17.0.8:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VoteBurrito
rsp id=0:460 src=172.17.0.9:45244 dst=172.17.0.8:8080 :status=200 latency=767µs
end id=0:460 src=172.17.0.9:45244 dst=172.17.0.8:8080 grpc-status=OK duration=18µs response-length=5B
req id=0:461 src=172.17.0.9:45244 dst=172.17.0.8:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VoteDog
rsp id=0:461 src=172.17.0.9:45244 dst=172.17.0.8:8080 :status=200 latency=693µs
end id=0:461 src=172.17.0.9:45244 dst=172.17.0.8:8080 grpc-status=OK duration=10µs response-length=5B
req id=0:462 src=172.17.0.9:45244 dst=172.17.0.8:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VotePoop
```

我们看看是不是可以在其中发现点什么。我们注意到日志中有一些`grpc-status=Unknown`，这表示GRPC请求失败。

我们接下来看看这一问题的来由。再次运行`tap`命令，过滤出其中的`Unknown`：

```
conduit tap deploy emojivoto/voting | grep Unknown -B 2

req id=0:212 src=172.17.0.8:58326 dst=172.17.0.10:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VotePoop
rsp id=0:212 src=172.17.0.8:58326 dst=172.17.0.10:8080 :status=200 latency=360µs
end id=0:212 src=172.17.0.8:58326 dst=172.17.0.10:8080 grpc-status=Unknown duration=0µs response-length=0B
--
req id=0:215 src=172.17.0.8:58326 dst=172.17.0.10:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VotePoop
rsp id=0:215 src=172.17.0.8:58326 dst=172.17.0.10:8080 :status=200 latency=414µs
end id=0:215 src=172.17.0.8:58326 dst=172.17.0.10:8080 grpc-status=Unknown duration=0µs response-length=0B
--
```

这样看到，所有的`grpc-status=Unknown`都来源于`VotePoop`这一端点。我们可以使用`tap`命令的参数，来关注这一端点的输出：

```
conduit tap deploy emojivoto/voting --path /emojivoto.v1.VotingService/VotePoop

req id=0:264 src=172.17.0.8:58326 dst=172.17.0.10:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VotePoop
rsp id=0:264 src=172.17.0.8:58326 dst=172.17.0.10:8080 :status=200 latency=696µs
end id=0:264 src=172.17.0.8:58326 dst=172.17.0.10:8080 grpc-status=Unknown duration=0µs response-length=0B
req id=0:266 src=172.17.0.8:58326 dst=172.17.0.10:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VotePoop
rsp id=0:266 src=172.17.0.8:58326 dst=172.17.0.10:8080 :status=200 latency=667µs
end id=0:266 src=172.17.0.8:58326 dst=172.17.0.10:8080 grpc-status=Unknown duration=0µs response-length=0B
req id=0:270 src=172.17.0.8:58326 dst=172.17.0.10:8080 :method=POST :authority=voting-svc.emojivoto:8080 :path=/emojivoto.v1.VotingService/VotePoop
rsp id=0:270 src=172.17.0.8:58326 dst=172.17.0.10:8080 :status=200 latency=346µs
end id=0:270 src=172.17.0.8:58326 dst=172.17.0.10:8080 grpc-status=Unknown duration=0µs response-length=0B
```

这样就看到，所有`VotePoop`的请求都失败了。当我们尝试给💩投票时候会发生什么？在[指南的第五步](../getting-started/#step-five)中打开演示应用。

现在点击💩来给他投票：

![Demo application 💩 page](images/emojivoto-poop.png)

演示应用在投票给 💩 的时候就会出错。这样我们就知道错误来自何处了。接下来就可以扎进日志和代码，来研究具体故障原因了。在Conduit的未来版本中，我们甚至可以通过对路由规则的控制来修改某一端点被调用时候的行为。
