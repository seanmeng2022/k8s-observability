# k8s-observability
## k8s events
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
