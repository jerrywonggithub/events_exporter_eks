# Event Exporter in EKS

>场景：监控 EKS 集群下的 event 日志并告警通知

>方案：安装第三方插件 event exporter，监听event事件，并根据不同类别（Normal、Warning）事件输出到不同目的端 

>方案参考：https://github.com/opsgenie/kubernetes-event-exporter

### 架构图

![arc](https://github.com/jerrywonggithub/events_exporter_eks/blob/main/capture/arc.png)

### 创建 Namespace、ServiceAccount、ClusterRolebinding

>event_role.yaml

```
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: monitoring
  name: event-exporter
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: event-exporter
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    namespace: monitoring
    name: event-exporter
```

### 创建 event exporter Config Map

>event_cm.yaml

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: event-exporter-cfg
  namespace: monitoring
data:
  config.yaml: |
    logLevel: debug
    logFormat: json
    route:      
      routes:
        - match:
            - receiver: "es"
        - drop:
            - namespace: "*"
            - type: "Normal"
          match:
            - receiver: "sns"
    receivers:
      - name: "es"
        elasticsearch:
          hosts:
            - https://vpc-jerry-demo-aos-tjn76hgdpuspm3jypdufmvlpey.us-west-2.es.amazonaws.com:443
          index: kube-events
          indexFormat: "kube-events-{2006-01-02}"
          username: # optional
          password: # optional
          useEventID: true
          deDot: false
          layout: # optional
          tls: # optional
      - name: "sns"
        sns:
          topicARN: "arn:aws:sns:us-west-2:032304891690:serverlessrepo-DingTalk-Notifier-DingTalkNotifierSNSTopic-1NR0VOY2V678L"
          region: "us-west-2"
          layout:
            eventType: "kube-event"
            createdAt: "{{ .GetTimestampMs }}"
            details:
              message: "{{ .Message }}"
              reason: "{{ .Reason }}"
              tip: "{{ .Type }}"
              count: "{{ .Count }}"
              kind: "{{ .InvolvedObject.Kind }}"
              name: "{{ .InvolvedObject.Name }}"
              namespace: "{{ .Namespace }}"
              component: "{{ .Source.Component }}"
              host: "{{ .Source.Host }}"
              labels: "{{ toJson .InvolvedObject.Labels}}"
  
```

### 创建 deployment

>event_deployment.yaml

```
apiVersion: apps/v1

          volumeMounts:
            - mountPath: /data
              name: cfg
      volumes:
        - name: cfg
          configMap:
            name: event-exporter-cfg
  selector:
    matchLabels:
      app: event-exporter
      version: v1
```

### 部署

```
kubectl create -f event_role.yaml
kubectl create -f event_cm.yaml
kubectl create -f event_deployment.yaml
```

### 验证

#### 检查pod运行状态为 running

```
kubectl get pod -n monitoring
NAME                             READY   STATUS    RESTARTS   AGE
event-exporter-d88d4f8f4-zh5ng   1/1     Running   0          131m
```

#### 在 OpenSearch 上检查 index，验证event是否已正常打到 Opensearch

![index](https://github.com/jerrywonggithub/events_exporter_eks/blob/main/capture/index.png)
![discover](https://github.com/jerrywonggithub/events_exporter_eks/blob/main/capture/opensearch.png)
