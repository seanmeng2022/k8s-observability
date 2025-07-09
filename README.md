# k8s-observability
## Log
### k8s events
Kubernetes 事件提供了丰富的信息源，可用于监控应用程序和集群状态、响应故障并执行诊断。事件通常表示某种状态变化。示例包括 Pod 创建、添加副本和调度资源。每个事件都包含一个 type 字段，该字段设置为 Normal 或 Warning 以指示成功或失败。
如果您曾经对资源运行过 kubectl describe 命令，那么您已经使用过 Kubernetes 事件。如下所示，kubectl describe 输出的最后一部分显示了与该资源相关的 Kubernetes 事件。
```
kubectl describe pod nginx
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  5s    default-scheduler  Successfully assigned default/nginx to ip-10-42-179-183.us-west-2.compute.internal
  Normal  Pulling    4s    kubelet            Pulling image "nginx"
  Normal  Pulled     4s    kubelet            Successfully pulled image "nginx" in 627.545722ms (627.553403ms including waiting)
  Normal  Created    4s    kubelet            Created container nginx
  Normal  Started    3s    kubelet            Started container nginx
```
Kubernetes 事件会持续生成，但在集群内仅保留一小时。这个保留期与 Kubernetes 上游默认事件生存时间(TTL)60分钟一致。OpenSearch 提供了一个持久化存储，简化了这些事件的收集、分析和可视化。
下图概述了本节的设置。kubernetes-events-exporter 将部署在 opensearch-exporter 命名空间中，用于将事件转发到 OpenSearch 域。事件存储在 OpenSearch 的 eks-kubernetes-events 索引中。我们之前加载的 OpenSearch 仪表板用于可视化这些事件。
<img width="794" alt="image" src="https://github.com/user-attachments/assets/eb6f7bf0-f1dc-4479-9910-e719b359713e" />

部署 Kubernetes 事件导出器并配置它以将事件发送到我们的 OpenSearch 域。基本配置可在此处获取(https://github.com/aws-samples/eks-workshop-v2/blob/stable/manifests/modules/observability/opensearch/config/events-exporter-values.yaml)。
```
helm install events-to-opensearch \
    oci://registry-1.docker.io/bitnamicharts/kubernetes-event-exporter \
    --namespace opensearch-exporter --create-namespace \
    -f ~/environment/eks-workshop/modules/observability/opensearch/config/events-exporter-values.yaml \
    --set="config.receivers[0].opensearch.username"="$OPENSEARCH_USER" \
    --set="config.receivers[0].opensearch.password"="$OPENSEARCH_PASSWORD" \
    --set="config.receivers[0].opensearch.hosts[0]"="https://$OPENSEARCH_HOST" \
    --wait

kubectl get pods -n opensearch-exporter
```

### Pod Logging
容器化应用程序应将其日志输出到 stdout 和 stderr。这在 Kubernetes 中也被认为是最佳实践，集群级日志收集系统正是基于此构建的。
Kubernetes 日志架构定义了三个不同的级别：
* 基础日志：使用 kubectl 获取 Pod 日志的能力（例如，kubectl logs myapp - 其中 myapp 是我集群中运行的 Pod）。
* 节点级日志：容器引擎从应用程序的标准输出 (stdout) 和标准错误输出 (stderr) 捕获日志，并将其写入日志文件。
* 集群级日志：基于节点级日志；每个节点上运行一个日志捕获代理。该代理收集本地文件系统上的日志，并将其发送到像 OpenSearch 这样的集中式日志目标。代理收集两种类型的日志：
  * 由节点上的容器引擎捕获的容器日志
  * 系统日志

Kubernetes 本身并不提供收集和存储日志的原生解决方案。它会配置容器运行时，将日志以 JSON 格式保存在本地文件系统上。容器运行时（例如 Docker）会将容器的标准输出 (stdout) 和标准错误输出 (stderr) 流重定向到日志驱动程序。在 Kubernetes 中，容器日志会写入节点上的 /var/log/pods/*.log 文件。Kubelet 和容器运行时会将自己的日志写入 /var/logs 文件，如果操作系统安装了 systemd，则会写入 journald 文件。然后，像 Fluentd 这样的集群范围的日志收集系统可以在节点上跟踪这些日志文件，并将日志发送出去进行保留。这些日志收集系统通常以 DaemonSet 的形式在工作节点上运行。

Fluent Bit 是一款轻量级日志处理器和转发器，可让您从不同来源收集数据和日志，使用筛选器对其进行丰富，并将它们发送到多个目标，例如 CloudWatch、Kinesis Data Firehose、Kinesis Data Streams 和 Amazon OpenSearch Service。
下图概述了本节的设置。Fluent Bit 将部署在 opensearch-exporter 命名空间中，并配置为将 Pod 日志转发到 OpenSearch 域。Pod 日志存储在 OpenSearch 的 eks-pod-logs 索引中。我们之前加载的 OpenSearch 仪表板用于检查 Pod 日志。
<img width="808" alt="image" src="https://github.com/user-attachments/assets/f9d0426e-f38b-40cc-8905-d6bfd7554531" />

将 Fluent Bit 部署为守护进程集，并将其配置为将 Pod 日志发送到 OpenSearch 域。基本配置可在此处获取（https://github.com/aws-samples/eks-workshop-v2/blob/stable/manifests/modules/observability/opensearch/config/fluentbit-values.yaml）。
```
helm repo add eks https://aws.github.io/eks-charts
helm upgrade fluentbit eks/aws-for-fluent-bit --install \
    --namespace opensearch-exporter --create-namespace \
    -f ~/environment/eks-workshop/modules/observability/opensearch/config/fluentbit-values.yaml \
    --set="opensearch.host"="$OPENSEARCH_HOST" \
    --set="opensearch.awsRegion"=$AWS_REGION \
    --set="opensearch.httpUser"="$OPENSEARCH_USER" \
    --set="opensearch.httpPasswd"="$OPENSEARCH_PASSWORD" \
    --wait
kubectl get daemonset -n opensearch-exporter
```
## Metrics
为了从 Amazon EKS 集群收集指标，我们将部署一个 OpenTelemetryCollector 自定义资源。对应的operator会照用户在 OpenTelemetryCollector 资源配置中指定的方式部署 ADOT 收集器实例。
部署ADOT resource需有对应的权限：
```
ec2-user:~/environment:$ kubectl kustomize ~/environment/eks-workshop/modules/observability/oss-metrics/adot \
  | envsubst | kubectl apply -f-
serviceaccount/adot-collector created
clusterrole.rbac.authorization.k8s.io/otel-prometheus-role created
clusterrolebinding.rbac.authorization.k8s.io/otel-prometheus-role-binding created
opentelemetrycollector.opentelemetry.io/adot created
```
其中，cluster role如下：
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-prometheus-role
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
      - nodes/proxy
      - services
      - endpoints
      - pods
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - nonResourceURLs:
      - /metrics
    verbs:
      - get

```
对应role权限如下：
```
~
$
aws iam list-attached-role-policies \
  --role-name $EKS_CLUSTER_NAME-adot-collector | jq .
{
  "AttachedPolicies": [
    {
      "PolicyName": "AmazonPrometheusRemoteWriteAccess",
      "PolicyArn": "arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess"
    }
  ]
}
```
对应ADOT Config如下：
```
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: adot
  namespace: other
spec:
  image: public.ecr.aws/aws-observability/aws-otel-collector:v0.40.0
  mode: deployment
  serviceAccount: adot-collector
  config:
    receivers:
      prometheus:
        config:
          global:
            scrape_interval: 60s
            scrape_timeout: 15s
            external_labels:
              cluster: ${EKS_CLUSTER_NAME}
              account_id: ${AWS_ACCOUNT_ID}
              region: ${AWS_REGION}
          scrape_configs:
            - job_name: "kubernetes-kubelet"
              scrape_interval: 60s
              scrape_timeout: 15s
              scheme: https
              tls_config:
                ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                insecure_skip_verify: true
              bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
              kubernetes_sd_configs:
                - role: node
              relabel_configs:
                - action: labelmap
                  regex: __meta_kubernetes_node_label_(.+)
                - target_label: __address__
                  replacement: kubernetes.default.svc.cluster.local:443
                - source_labels: [__meta_kubernetes_node_name]
                  regex: (.+)
                  target_label: __metrics_path__
                  replacement: /api/v1/nodes/$${1}/proxy/metrics
            - job_name: "kubelet"
              scheme: https
              tls_config:
                ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                insecure_skip_verify: true
              bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
              kubernetes_sd_configs:
                - role: node
              relabel_configs:
                - action: labelmap
                  regex: __meta_kubernetes_node_label_(.+)
                - target_label: __address__
                  replacement: kubernetes.default.svc.cluster.local:443
                - source_labels: [__meta_kubernetes_node_name]
                  regex: (.+)
                  target_label: __metrics_path__
                  replacement: /api/v1/nodes/$${1}/proxy/metrics/cadvisor
            - job_name: "kubernetes-pods"
              honor_labels: true

              kubernetes_sd_configs:
                - role: pod

              relabel_configs:
                - source_labels:
                    [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
                  action: keep
                  regex: true
                - source_labels:
                    [__meta_kubernetes_pod_annotation_prometheus_io_scrape_slow]
                  action: drop
                  regex: true
                - source_labels:
                    [__meta_kubernetes_pod_annotation_prometheus_io_scheme]
                  action: replace
                  regex: (https?)
                  target_label: __scheme__
                - source_labels:
                    [__meta_kubernetes_pod_annotation_prometheus_io_path]
                  action: replace
                  target_label: __metrics_path__
                  regex: (.+)
                - action: labelmap
                  regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
                  replacement: __param_$$1
                - action: labelmap
                  regex: __meta_kubernetes_pod_label_(.+)
                - source_labels: [__meta_kubernetes_namespace]
                  action: replace
                  target_label: namespace
                - source_labels: [__meta_kubernetes_pod_name]
                  action: replace
                  target_label: pod
                - source_labels: [__meta_kubernetes_pod_phase]
                  regex: Pending|Succeeded|Failed|Completed
                  action: drop
    exporters:
      prometheusremotewrite:
        endpoint: ${AMP_ENDPOINT}api/v1/remote_write
        auth:
          authenticator: sigv4auth
      logging:
        loglevel: info
    extensions:
      sigv4auth:
        region: ${AWS_REGION}
        service: aps
      health_check: {}
      pprof:
        endpoint: :1888
      zpages:
        endpoint: :55679
    service:
      extensions: [pprof, zpages, health_check, sigv4auth]
      pipelines:
        metrics:
          receivers: [prometheus]
          processors: []
          exporters: [logging, prometheusremotewrite]
```
这将配置一个具有以下结构的 OpenTelemetry 管道：

* 接收器
  * Prometheus 接收器，用于从公开 Prometheus 端点的目标中抓取指标
* 处理器
  * 不涉及
* 导出器
  * Prometheus 远程写入导出器，用于将指标发送到 Prometheus 远程写入端点（例如 AMP）


确认部署情况：
```
ec2-user:~/environment:$ kubectl get pods -n other
NAME                              READY   STATUS    RESTARTS   AGE
adot-collector-5486949687-5m9wf   1/1     Running   0          9m52s
```
