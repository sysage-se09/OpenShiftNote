# OpenShift 4.x Infra Node Setting

透過 taint、toleration 和 nodeSelector 來達成。

## Infra Node Setting

### Node

新增 Infra 角色至 Node
```
oc label node <node-name> node-role.kubernetes.io/infra=
```

從 Node 移除 Worker 角色
```
oc label node <node-name> node-role.kubernetes.io/worker- 
```

建立 Infrastructure Machine Config Pool (MCP)，並將設定套用至 Infra Node 上
```
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfigPool
metadata:
  name: infra
spec:
  machineConfigSelector:
    matchExpressions:
      - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,infra]}
  nodeSelector:
    matchLabels:
      node-role.kubernetes.io/infra: ""
```

使用上面的 mcp 設定建立物件
```
oc create -f infra-mcp.yaml
```

檢查是否建立完成
```
oc get mc
```
```
oc get mcp
```

上 taint 確保 Infra Node 上只能運行跟 infrastructure 相關的元件
```
oc adm taint nodes -l node-role.kubernetes.io/infra infra=reserved:NoSchedule infra=reserved:NoExecute
```

將 infrastructure 元件移至 Infra Node 上記得要加上以下 toleration
```
spec:
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
    tolerations:
    - effect: NoSchedule
      key: infra
      value: reserved
    - effect: NoExecute
      key: infra
      value: reserved
```

### Router

```
oc patch ingresscontroller/default -n  openshift-ingress-operator  --type=merge -p '{"spec":{"nodePlacement": {"nodeSelector": {"matchLabels": {"node-role.kubernetes.io/infra": ""}},"tolerations": [{"effect":"NoSchedule","key": "infra","value": "reserved"},{"effect":"NoExecute","key": "infra","value": "reserved"}]}}}'
```

```
oc patch ingresscontroller/default -n openshift-ingress-operator --type=merge -p '{"spec":{"replicas": 3}}'
```

### Registry

```
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"nodeSelector": {"node-role.kubernetes.io/infra": ""},"tolerations": [{"effect":"NoSchedule","key": "infra","value": "reserved"},{"effect":"NoExecute","key": "infra","value": "reserved"}]}}'
```

### Monitoring

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: infra
        value: reserved
        effect: NoSchedule
      - key: infra
        value: reserved
        effect: NoExecute
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: infra
        value: reserved
        effect: NoSchedule
      - key: infra
        value: reserved
        effect: NoExecute
    prometheusOperator:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: infra
        value: reserved
        effect: NoSchedule
      - key: infra
        value: reserved
        effect: NoExecute
    grafana:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: infra
        value: reserved
        effect: NoSchedule
      - key: infra
        value: reserved
        effect: NoExecute
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: infra
        value: reserved
        effect: NoSchedule
      - key: infra
        value: reserved
        effect: NoExecute
    kubeStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: infra
        value: reserved
        effect: NoSchedule
      - key: infra
        value: reserved
        effect: NoExecute
    telemeterClient:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: infra
        value: reserved
        effect: NoSchedule
      - key: infra
        value: reserved
        effect: NoExecute
    openshiftStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      tolerations:
      - key: infra
        value: reserved
        effect: NoSchedule
      - key: infra
        value: reserved
        effect: NoExecute
```

```
oc create -f cluster-monitoring-configmap.yaml
```

### Logging

```
oc edit ClusterLogging instance
```

```
apiVersion: logging.openshift.io/v1
kind: ClusterLogging

....

spec:
  collection:
    logs:
      fluentd:
        resources: null
      type: fluentd
  curation:
    curator:
      nodeSelector: 
          node-role.kubernetes.io/infra: ''
      resources: null
      schedule: 30 3 * * *
    type: curator
  logStore:
    elasticsearch:
      nodeCount: 3
      nodeSelector: 
          node-role.kubernetes.io/infra: ''
      redundancyPolicy: SingleRedundancy
      resources:
        limits:
          cpu: 500m
          memory: 16Gi
        requests:
          cpu: 500m
          memory: 16Gi
      storage: {}
    type: elasticsearch
  managementState: Managed
  visualization:
    kibana:
      nodeSelector: 
          node-role.kubernetes.io/infra: '' 
      proxy:
        resources: null
      replicas: 1
      resources: null
    type: kibana

....
```

## Infra Node Resources Limit

### Router
:::info
可以直接修改 deployment
:::

修改 Deployment 設定
```
oc edit deployment router-default -n openshift-ingress
```
```yaml
...
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
```

### Registry
:::info
必須先將 Imageregistry operator 設定為 Unmanaged 狀態
* 若設定為 Unmanaged 的話，則 image prune 功能會設定為 false 且若 operator 有更新時必須手動進行更新
:::

修改 Imageregistry operator 狀態為 Unmanaged
```
oc edit config cluster -n openshift-registry-operator
```
```yaml
...
spec:
  ...
  managementState: Managed
```

修改 image registry deployment
```
oc edit deployment image-registry -n openshift-image-registry
```
```yaml
...
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
```

### Monitoring
:::info
可以透過修改 custom resource 檔案來調整，主要調整的組件有以下兩個
1. Prometheus
2. AlertManger
:::

Prometheus
```
oc edit prometheus k8s -n openshift-monitoring
```
> 有 9 個地方可以修改

AlertManger
```
oc edit alertmanager main -n openshift-monitoring
```
> 有 2 個地方可以修改


### Logging
:::info
可以透過修改 custom resource 檔案來調整，主要調整的組件有以下三個
1. Elasticsearch
2. Kibana
3. Fluentd
:::

修改 custom resource，並找到以下三個部分進行修改
```
oc edit ClusterLogging instance -n openshift-logging
```

Elasticsearch
```yaml
spec:
  ...
  logStore:
    type: "elasticsearch"
    elasticsearch:
      resources: 
        limits:
          memory: 16Gi
        requests:
          cpu: 500m
          memory: 16Gi
```

Kibana
```yaml
spec:
  ...
  visualization:
    type: "kibana"
    kibana:
      replicas:
    resources:  
      limits:
        memory: 1Gi
      requests:
        cpu: 500m
        memory: 1Gi
    proxy:  
      resources:
        limits:
          memory: 100Mi
        requests:
          cpu: 100m
          memory: 100Mi
```

Fluentd
```yaml
spec:
  ...
  collection:
    logs:
      fluentd:
        resources:
          limits: 
            memory: 736Mi
          requests:
            cpu: 100m
            memory: 736Mi
```

## Infra Component Information

### Router

相關元件位於 openshift-ingress project 中，可用以下指令查詢
```
oc get po -n openshift-ingress
```


### Registry

相關元件位於 openshift-image-registry project 中，可用以下指令查詢
```
oc get po -n openshift-image-registry
```

### Monitoring

相關元件位於 openshift-monitoring project 中，可用以下指令查詢
```
oc get po -n openshift-monitoring
```

### Logging

相關元件位於 openshift-logging project 中，可用以下指令查詢
```
oc get po -n openshift-logging
```


## Reference
* [Configuring Infrastructure Nodes with Taint in OpenShift 4](https://access.redhat.com/solutions/5034771)
* [Moving the cluster logging resources](https://docs.openshift.com/container-platform/4.5/machine_management/creating-infrastructure-machinesets.html#infrastructure-moving-logging_creating-infrastructure-machinesets)
* [Recommended host practices](https://docs.openshift.com/container-platform/4.5/scalability_and_performance/recommended-host-practices.html)

---

###### tags: `OpenShift`