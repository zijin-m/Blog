# pod 无法访问集群 ingress slb 绑定的公网域名

`pod` 请求 `slb` 域名失败的问题总结文档。

# 问题背景

在使用 `cert-manager` 的 `http-01` 方式自动生成证书的时候发现 `Certificate` 一直是 `pending` 状态，故进行检查 `Challenges` 的状态，发现如下错误：

```
$ kubectl describe challenge example-com-2745722290-4391602865-0
...
Waiting for http-01 challenge propagation: failed to perform self check GET request ‘http://example-com/.well-known/acme-challenge/Rjx7CiiA8GdIhXWRqa_s5XVswaHlD3y0XGMa1LJIIps’: Get http://example-com/.well-known/acme-challenge/Rjx7CiiA8GdIhXWRqa_s5XVswaHlD3y0XGMa1LJIIps: dial tcp 120.55.227.13:80: connect: connection refused
...
```

从错误信息上看出来是无法访问 `http://example-com/.well-known/acme-challenge/Rjx7CiiA8GdIhXWRqa_s5XVswaHlD3y0XGMa1LJIIps` ，这个链接是 `cert-manager` 自动生成用来验证域名使用权的，访问正常会返回一串 `token` 字符串，验证成功就可以颁发证书，但是如果无法访问也就无法创建证书了，难道是链接访问出错？故在自己的机器上尝试请求。

```bash
$ curl http://example-com/.well-known/acme-challenge/Rjx7CiiA8GdIhXWRqa_s5XVswaHlD3y0XGMa1LJIIps
```

但是上述步骤是可以正常返回拿到 `200` 返回码和 证书验证 `token` 串的。

那么为何 `cert-manager` 进行验证的时候会出错呢，和本地访问的区别在哪？ `cer-manager` 是通过集群内部的 `pod` 进行访问的，那么会不会和这个有关系呢？

启动临时 pod 进行验证：
```bash
 $ kubectl run -it --rm --restart=Never curl --image=radial/busyboxplus:curl sh
 $ curl http://example-com/.well-known/acme-challenge/Rjx7CiiA8GdIhXWRqa_s5XVswaHlD3y0XGMa1LJIIps
```
得到错误：`curl: (7) Failed to connect to example-com port 80: Connection refused` 。

结论：集群内部无法正常访问 `http://example-com` ，进一步发现只要是和集群 `slb` 进行绑定的域名都无法访问，如 `http://abc.example-com` 等。

# 原因分析

通过对相关内部 `pod` 无法访问集群域名的一番查找，将问题锁定在了 `externalTrafficPolicy` 这个参数下，这个参数一般用于决定是否保留客户端 IP，说明如下：

* `service.spec.externalTrafficPolicy ` - 表示此服务是否希望将外部流量路由到节点本地或集群范围的端点。有两个可用选项：Cluster（默认）和 Local。Cluster 隐藏了客户端源 IP，可能导致第二跳到另一个节点，但具有良好的整体负载分布。Local 保留客户端源 IP 并避免 LoadBalancer 和 NodePort 类型服务的第二跳，但存在潜在的不均衡流量传播风险。
* `service.spec.healthCheckNodePort` - 指定服务的 healthcheck nodePort（数字端口号）。如果未指定，则 serviceCheckNodePort 由服务 API 后端使用已分配的 nodePort 创建。如果客户端指定，它将使用客户端指定的 nodePort 值。仅当 type 设置为 LoadBalancer 并且 externalTrafficPolicy 设置为 Local 时才生效

查看了 `ingress service` 的 `spec` 字段，果然 `externalTrafficPolicy` 为 `Local` ，这就导致了不会跳到别的节点，那么这是什么意思呢，具体来说，设置 `service.spec.externalTrafficPolicy` 的值为 `Local` ，请求就只会被代理到本地 endpoints 而不会被转发到其它节点。这样就保留了最初的源 IP 地址。如果没有本地 endpoints，发送到这个节点的数据包将会被丢弃。这样在应用到数据包的任何包处理规则下，你都能依赖这个正确的 source-ip 使数据包通过并到达 endpoint。

举例来说，如果你的集群有两个节点，node1 上有一个 `pod` 提供服务，你使用了 `NodePort` （这个规则同样适用于 `LoadBalancer`） 类型的 `svc` 暴露出这个服务。如果你分别对两个 node 进行请求，那么，你只会从 endpoint pod 运行的那个节点得到了一个回复，这个回复有正确的客户端 IP。

发生了什么：

* 客户端发送数据包到 node2:nodePort，它没有任何 endpoints
数据包被丢弃
* 客户端发送数据包到 node1:nodePort，它*有*endpoints
* node1 使用正确的源 IP 地址将数据包路由到 endpoint

用图表示：

```
        client
       ^ /   \
      / /     \
     / v       X
   node 1     node 2
    ^ |
    | |
    | v
 endpoint
```

# 问题总结

那么 `cert-manager` 为什么无法访问集群域名呢？原因在于集群域名对应的 `ip` 是 `LoadBalancer` 类型的 `svc` 的 `ip` ， 而 `svc` 对应的 `pod`（在 node1 上） 和 `cert-manager` 的 `pod`（在 node2 上） 不在一个 `node` 上，那么按照上面的理论，不会进行节点的转发，数据包被丢弃，请求自然失败。

发什么什么:

* node1 上的 pod 执行 curl http://example-com
* dns 解析返回 `xx.xx.xx.xx`  ip 地址，注意这个 ip 地址被 `svc` 使用，认为是集群内部地址
* 发送数据包到`xx.xx.xx.xx` ，node1 接收到数据包，没有任何 endpoints ，数据包被丢弃（因为不会转发到 node2 上去 ）

# 解决方法

* 将 `externalTrafficPolicy` 设置为 `Cluster` ，使用一种和后端之间约定的协议来和真实的客户端 IP 通信，例如 HTTP X-FORWARDED-FOR 头，或者 proxy 协议 。
* 将 `Ingress Controller` 的 `pod` 在每个节点部署，或者使用 `DaemonSet` 方式部署。


# 参考文档

* [启用IPVS的K8S集群无法从Pod经外部访问自己的排障](https://chanjarster.github.io/post/k8s/k8s-ipvs-cannot-access-self-from-cluster/)
* [k8s-svc-nodeport](https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-nodeport)

