# RabbitMQ 集群部署方式
通过官方镜像 RabbitMQ Docker Image 和 rabbitmq-peer-discovery-k8s 插件进行集群部署。

# 命名空间

将 rabbitmq 的资源都放在 rabbitmq 命名空间内。

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rabbitmq
```
# 配置

通过配置 configMap 将配置文件挂载到 rabbitmq 容器内 。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-config
  namespace: rabbitmq
data:
  enabled_plugins: |
      [rabbitmq_management,rabbitmq_peer_discovery_k8s].
  rabbitmq.conf: |
      ## Cluster formation. See https://www.rabbitmq.com/cluster-formation.html to learn more.
      cluster_formation.randomized_startup_delay_range.min = 0
      cluster_formation.randomized_startup_delay_range.max = 1
      cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
      ## Service name is rabbitmq by default but can be overridden using the cluster_formation.k8s.service_name key if needed
      cluster_formation.k8s.service_name = rabbitmq-internal
      ## It is possible to append a suffix to peer hostnames returned by Kubernetes using cluster_formation.k8s.hostname_suffix
      cluster_formation.k8s.hostname_suffix = .rabbitmq-internal.rabbitmq.svc.cluster.local
      ## Should RabbitMQ node name be computed from the pod's hostname or IP address?
      ## IP addresses are not stable, so using [stable] hostnames is recommended when possible.
      ## Set to "hostname" to use pod hostnames.
      ## When this value is changed, so should the variable used to set the RABBITMQ_NODENAME
      ## environment variable.
      cluster_formation.k8s.address_type = hostname
      ## How often should node cleanup checks run?
      cluster_formation.node_cleanup.interval = 30
      ## Set to false if automatic removal of unknown/absent nodes
      ## is desired. This can be dangerous, see
      ##  * https://www.rabbitmq.com/cluster-formation.html#node-health-checks-and-cleanup
      ##  * https://groups.google.com/forum/#!msg/rabbitmq-users/wuOfzEywHXo/k8z_HWIkBgAJ
      cluster_formation.node_cleanup.only_log_warning = true
      cluster_partition_handling = autoheal
      ## See https://www.rabbitmq.com/ha.html#master-migration-data-locality
      queue_master_locator=min-masters
      ## This is just an example.
      ## This enables remote access for the default user with well known credentials.
      ## Consider deleting the default user and creating a separate user with a set of generated
      ## credentials instead.
      ## Learn more at https://www.rabbitmq.com/access-control.html#loopback-users
      loopback_users.guest = false
      ## https://www.rabbitmq.com/memory.html#configuring-threshold
      vm_memory_high_watermark.relative = 0.6
```

# 密钥

通过 secrets 将 erlang-cookie 和默认用户信息写入到环境变量中。

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq-secret
  namespace: rabbitmq
type: Opaque
data:
  RABBITMQ_ERLANG_COOKIE: MTIzajE5dWVkYXM3ZGFkODEwMjNqMTM5ZGph
  RABBITMQ_DEFAULT_USER: dXNlcg==
  RABBITMQ_DEFAULT_PASS: cGFzc3dvcmQ=
```
# RBAC

rabbitmq-peer-discovery 需要 rabc 权限来获取 endpoints  信息来做集群节点的自动发现。

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rabbitmq
  namespace: rabbitmq
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rabbitmq-peer-discovery-rbac
  namespace: rabbitmq
rules:
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: rabbitmq-peer-discovery-rbac
  namespace: rabbitmq
subjects:
- kind: ServiceAccount
  name: rabbitmq
  namespace: rabbitmq
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: rabbitmq-peer-discovery-rbac
```
# 服务

定义 headless service 作为 statefulset 的服务入口。
```yaml
kind: Service
apiVersion: v1
metadata:
  namespace: rabbitmq 
  name: rabbitmq-internal
  labels:
    app: rabbitmq
spec:
  clusterIP: None
  ports:
    - name: epmd
      protocol: TCP
      port: 4369
    - name: amqp
      protocol: TCP
      port: 5672
    - name: amqp-tls
      protocol: TCP
      port: 5671
    - name: http
      protocol: TCP
      port: 15672
    - name: inter-node-cli
      protocol: TCP
      port: 25672
  selector:
    app: rabbitmq 
```
# 有状态应用部署

按照官方集群部署的推荐方式使用 StatefulSet 方式部署，使用动态存储卷保存数据。

```yaml
apiVersion: apps/v1
# See the Prerequisites section of https://www.rabbitmq.com/cluster-formation.html#peer-discovery-k8s.
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: rabbitmq
spec:
  serviceName: rabbitmq-internal
  # Three nodes is the recommended minimum. Some features may require a majority of nodes
  # to be available.
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      serviceAccountName: rabbitmq
      terminationGracePeriodSeconds: 10
      nodeSelector:
        # Use Linux nodes in a mixed OS kubernetes cluster.
        # Learn more at https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-os
        kubernetes.io/os: linux
      initContainers:
        - name: fix-readonly-config
          image: busybox:1.31.1
          command:
            - sh
            - -c
            - cp /tmp/config/* /etc/rabbitmq;
          volumeMounts:
            - name: rabbitmq-config
              mountPath: /etc/rabbitmq
            - name: tmp-dir
              mountPath: /tmp/config
      containers:
        - name: rabbitmq
          image: rabbitmq:3.8.2
          # Learn more about what ports various protocols use
          # at https://www.rabbitmq.com/networking.html#ports
          ports:
            - name: epmd
              protocol: TCP
              containerPort: 4369
            - name: amqp
              protocol: TCP
              containerPort: 5672
            - name: amqp-tls
              protocol: TCP
              containerPort: 5671
            - name: http
              protocol: TCP
              containerPort: 15672
          livenessProbe:
            exec:
              # This is just an example. There is no "one true health check" but rather
              # several rabbitmq-diagnostics commands that can be combined to form increasingly comprehensive
              # and intrusive health checks.
              # Learn more at https://www.rabbitmq.com/monitoring.html#health-checks.
              #
              # Stage 2 check:
              command: ["rabbitmq-diagnostics", "status"]
            initialDelaySeconds: 60
            # See https://www.rabbitmq.com/monitoring.html for monitoring frequency recommendations.
            periodSeconds: 60
            timeoutSeconds: 15
          readinessProbe:
            exec:
              # This is just an example. There is no "one true health check" but rather
              # several rabbitmq-diagnostics commands that can be combined to form increasingly comprehensive
              # and intrusive health checks.
              # Learn more at https://www.rabbitmq.com/monitoring.html#health-checks.
              #
              # Stage 2 check:
              command: ["rabbitmq-diagnostics", "status"]
              # To use a stage 4 check:
              # command: ["rabbitmq-diagnostics", "check_port_connectivity"]
            initialDelaySeconds: 20
            periodSeconds: 60
            timeoutSeconds: 10
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: RABBITMQ_NODENAME
              value: rabbit@$(POD_NAME).rabbitmq-internal.$(POD_NAMESPACE).svc.cluster.local
            - name: RABBITMQ_USE_LONGNAME
              value: "true"
          envFrom:
            - secretRef:
                name: rabbitmq-secret
          volumeMounts:
            - name: rabbitmq-config
              mountPath: /etc/rabbitmq
            - name: rabbitmq-data
              mountPath: /var/lib/rabbitmq
      volumes:
        - name: rabbitmq-config
          emptyDir: {}
        - name: tmp-dir
          configMap:
            name: rabbitmq-config
  volumeClaimTemplates:
    - metadata:
        name: rabbitmq-data
        namespace: rabbitmq
        labels:
          app: rabbitmq
      spec:
        accessModes:
          - "ReadWriteOnce"
        resources:
          requests:
            storage: 20Gi
```

# 遇到的问题

### 只读文件错误

因为配置文件是通过 configmap 的方式挂载的，在 k8s 中，这种挂载方式是只读的，相关讨论在 Issue#62099 ，会导致类似 Read-only file system 的报错，相关讨论见 Issue#37 ，目前需要通过 initContainers 复制文件作为中转来避免只读报错。
### 节点无法互相发现加入

通过 rabbitmqctl cluster_status  的返回发现 rabbitmq server 和 rabbitmq-peer-discovery-k8s 使用的 nodename  不一致，需要添加额外环境变量 RABBITMQ_NODENAME 为 rabbit@$(POD_NAME).rabbitmq-internal.$(POD_NAMESPACE).svc.cluster.local ，让 rabbitmq  启动的时候使用同一个节点名，同时添加 RABBITMQ_USE_LONGNAME : true 。
# HA 配置

目前集群中 3 节点，设置 2 个镜像队列（包含 master ）来做高可用。
rabbitmqctl --erlang-cookie ${RABBITMQ_ERLANG_COOKIE} set_policy ha-exactly "^.*" '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'

# 参考

• https://github.com/rabbitmq/rabbitmq-peer-discovery-k8s

• https://zupzup.org/k8s-rabbitmq-cluster