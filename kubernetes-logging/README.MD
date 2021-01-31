## Подготовка.
```bash
mkdir kubernetes-logging && cd kubernetes-logging
```
### Кластер в GCP.
Создадим 
* 1 ноду типа n1-standard-2 в default-pool
* 3 ноды типа n1-standard-2 в infra-pool

Отключим логгирование k8s кластера, т.к оно может помешать нам в нашей работе.
```bash
--no-enable-cloud-logging #Отключить только логирование.
--no-enable-stackdriver-kubernetes #Отключить логгирование и мониторинг.
```
В нашем случае отключаем всё, добавив при создадии кластера параметр **--no-enable-stackdriver-kubernetes**

Создадим кластер **cluster-1** c 1 нодой **n1-standard-2** в пулл **default-pool**
```bash
gcloud beta container --project "digital-waters-300922" / 
clusters create "cluster-1" /
--zone "europe-west4-b" /
--no-enable-basic-auth / 
--cluster-version "1.17.14-gke.1600" /
--release-channel "regular" /
--machine-type "n1-standard-2" /
--image-type "COS" /
--disk-type "pd-standard" /
--disk-size "100" /
--metadata disable-legacy-endpoints=true /
--scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" /
--num-nodes "1" /
--enable-stackdriver-kubernetes /
--enable-ip-alias  / 
--network "projects/digital-waters-300922/global/networks/default" /
--subnetwork "projects/digital-waters-300922/regions/europe-west4/subnetworks/default" / 
--default-max-pods-per-node "110" / 
--no-enable-master-authorized-networks / 
--addons HorizontalPodAutoscaling,HttpLoadBalancing / 
--enable-autoupgrade /
--enable-autorepair /
--max-surge-upgrade 1 /
--max-unavailable-upgrade 0 /
--no-enable-stackdriver-kubernetes
```

Добавим в кластер **cluster-1** пулл **infra-pool** с 3 нодами **n1-standard-2**.

Мы хотим использовать **infra-pool** для инфраструктурных сервисов и присвоим им определенный [taint](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/) .Укажем это используя следующий параметр **--node-taints node-role=infra:NoSchedule** для нод, он запретит запуск подов без прописаниях ключей **tolerations**. 
```bash
gcloud beta container --project "digital-waters-300922" node-pools create "infra-pool" 
--cluster "cluster-1" /
--num-nodes 3 /
--zone "europe-west4-b" /
--machine-type "n1-standard-2" /
--image-type "COS" /
--disk-type "pd-standard"/
--disk-size "100" /
--metadata disable-legacy-endpoints=true /
--scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" /
--num-nodes "3" /
--enable-autoupgrade /
--enable-autorepair /
--max-surge-upgrade 1 /
--max-unavailable-upgrade 0 / 
--node-taints node-role=infra:NoSchedule 
```

Получим учетные данные для кластера.
```bash
gcloud container clusters get-credentials cluster-1
```

Провериим, что получилось.
```bash
> kubectl get nodes
NAME                                       STATUS   ROLES    AGE   VERSION
gke-cluster-1-default-pool-8ae49846-h3zn   Ready    <none>   66m   v1.17.14-gke.1600
gke-cluster-1-infra-pool-1e4e22a2-22gf     Ready    <none>   61m   v1.17.14-gke.1600
gke-cluster-1-infra-pool-1e4e22a2-63kh     Ready    <none>   61m   v1.17.14-gke.1600
gke-cluster-1-infra-pool-1e4e22a2-mxst     Ready    <none>   61m   v1.17.14-gke.1600
```

## Установка HipsterShop

Создадим **ns** и применим манифест.
```bash
kubectl create ns microservices-demo
kubectl apply -f https://raw.githubusercontent.com/1Psy/k8s_platform/main/kubernetes-logging/microservices-demo-without-resources.yaml -n microservices-demo
```
![pods_node](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-logging/pods_node.png)

# EFK Stack
# Elasticsearch 

Установим **Elasticsearch** из **helm** чарта.

Добавим репозиторий с чартами **Elasticsearch** **Kibana**.
```bash
> helm repo add elastic https://helm.elastic.co
"elastic" has been added to your repositories
```

Установим чарт c дефолтными параметрами указав только версию.
```
helm upgrade --install elasticsearch elastic/elasticsearch --version 7.10.2 --namespace observability --create-namespace
```

Проверим, что с нашим сервисом.
```bash
> kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE     IP           NODE
elasticsearch-master-0   0/1     Running   0          4m58s   10.80.0.29   gke-cluster-1-default-pool-8ae49846-h3zn   
elasticsearch-master-1   0/1     Pending   0          4m58s   <none>       <none>                                     
elasticsearch-master-2   0/1     Pending   0          4m58s   <none>       <none>                                     
```
Посмотрим что с master-1.
```bash
> kubectl describe pod elasticsearch-master-1
....
  Warning  FailedScheduling  54s (x4 over 3m26s)    default-scheduler  0/4 nodes are available: 1 Insufficient cpu, 3 node(s) had taints that the pod didn't tolerate.
```
Наши сервис попыталися запуститься в **default-pool**, но не смог из-за отсуствия ресурсов **cpu**.

Исправим это разрешив запуск подов в **infra-pool** на 3х нодах.

Для этого пропишем в [values](https://raw.githubusercontent.com/elastic/helm-charts/v7.10.2/elasticsearch/values.yaml) нашего сервиса пропишем ключ для запуска  на нодах с taint **node-role=infra:NoSchedule**.


Скачаем **values** **elasticsearch** и сохраним как **elasticsearch.values.yaml**.
```bash
wget -O elasticsearch.values.yaml https://raw.githubusercontent.com/elastic/helm-charts/v7.10.2/elasticsearch/values.yaml
```
Найдем там строку с **tolerations** и пропишем туда.
```bash
tolerations:
  - key: node-role
    operator: Equal
    value: infra
    effect: NoSchedule
```
Остальные поля удалим.

Обновим наше приложение.
```bash
helm upgrade --install elasticsearch elastic/elasticsearch --version 7.10.2 --namespace observability -f elasticsearch.values.yaml
```
Теперь **elasticsearch** может запускать на нодах из **infra-pool** и **default-pool**.

Оставим запуск только на нодах **infra-pool**, для этого добавим в **elasticsearch.values.yaml** **NodeSelector**, определяющий, на каких нодах запустить наши **pods**. 

Запросим все лейблы нод.
```bash
> kubectl get nodes --show-labels | grep infra
gke-cluster-1-infra-pool-1e4e22a2-22gf     Ready    <none>   135m   v1.17.14-gke.1600   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=n1-standard-2,beta.kubernetes.io/os=linux,cloud.google.com/gke-nodepool=infra-pool,cloud.google.com/gke-os-distribution=cos,cloud.google.com/machine-family=n1,failure-domain.beta.kubernetes.io/region=europe-west4,failure-domain.beta.kubernetes.io/zone=europe-west4-b,kubernetes.io/arch=amd64,kubernetes.io/hostname=gke-cluster-1-infra-pool-1e4e22a2-22gf,kubernetes.io/os=linux,node.kubernetes.io/instance-type=n1-standard-2,topology.kubernetes.io/region=europe-west4,topology.kubernetes.io/zone=europe-west4-b
```
Будем использовать лейбл **cloud.google.com/gke-nodepool=infra-pool**.
```bash
nodeSelector:
  cloud.google.com/gke-nodepool: infra-pool
```
После развертывания нашего сервиса с данным **nodeSelector**, все **pods** буду назначены на **node** с нашим лейблом.


## Другой, и, на самом деле, более гибкий способ осуществить задуманное - [nodeAffinity](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pods-nodes-using-node-affinity/)

Для этого нам пришлось бы прописать.
```bash
nodeAffinity: 
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cloud.google.com/gke-nodepool
            operator: In
            values:
            - infra-pool   
```
Обновим наше приложение.
```bash
helm upgrade --install elasticsearch elastic/elasticsearch --version 7.10.2 --namespace observability -f elasticsearch.values.yaml
```
Убедимя, что **elasticsearch** успешено развернулся.
```bash
> kubectl get pods -n observability -o wide -l chart=elasticsearch
NAME                     READY   STATUS    RESTARTS   AGE    IP          NODE                                   
elasticsearch-master-0   1/1     Running   0          2m4s   10.80.3.3   gke-cluster-1-infra-pool-1e4e22a2-63kh 
elasticsearch-master-1   1/1     Running   0          2m4s   10.80.1.4   gke-cluster-1-infra-pool-1e4e22a2-22gf
elasticsearch-master-2   1/1     Running   0          2m4s   10.80.2.3   gke-cluster-1-infra-pool-1e4e22a2-mxst
```

# nginx-ingress
Для работы с **kibana** нам нужен **nginx-ingress**, установим его на 3х нодах из **infra-pool**.

Добавим helm репозиторий.
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
Скачаем **values** **nginx-ingress** и сохраним как **nginx-ingress.values.yaml**.
```bash
wget -O nginx-ingress.values.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/helm-chart-3.22.0/charts/ingress-nginx/values.yaml
```
И заполним его.
```yaml
controller:
  replicaCount: 3

  tolerations:
    - key: node-role
      operator: Equal
      value: infra
      effect: NoSchedule

  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - nginx-ingress
          topologyKey: kubernetes.io/hostname

  nodeSelector: 
    cloud.google.com/gke-nodepool: infra-pool
```
Установим **nginx-ingress**.
```bash
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx --wait \
--namespace=nginx-ingress \
--create-namespace \
--version=3.22.0 \
-f nginx-ingress.values.yaml
```
Узнаем внешний ip LoadBalancer.
```bash
> kubectl get svc -n nginx-ingress
NAME                                               TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-ingress-nginx-controller             LoadBalancer   10.84.13.88   34.90.137.122   80:30858/TCP,443:30611/TCP   17m
nginx-ingress-ingress-nginx-controller-admission   ClusterIP      10.84.2.98    <none>          443/TCP                      17m
```
# Kibana

Скачаем **values** **Kibana** и сохраним как **Kibana.values.yaml**.
```bash
wget -O Kibana.values.yaml https://raw.githubusercontent.com/elastic/helm-charts/v7.10.2/kibana/values.yaml
```
И заполним его.
```yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
    # kubernetes.io/tls-acme: "true"
  path: /
  hosts:
    - kibana.34.90.137.122.xip.io
  tls: []
  #  - secretName: chart-example-tls
  #    hosts:
  #      - chart-example.local
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"

tolerations:
  - key: node-role
    operator: Equal
    value: infra
    effect: NoSchedule

nodeSelector:
  cloud.google.com/gke-nodepool: infra-pool

```
Выполним обновление релиза
```bash
helm upgrade --install kibana elastic/kibana --version 7.10.2 --namespace observability -f kibana.values.yaml
```
Kibana доступна.
http://kibana.34.90.137.122.xip.io
Но данных нет, т.к не установлен Fluent Bit.



# Fluent Bit

Добавим репозиторий с чартом **Fluent Bit**.
```bash
> helm repo add fluent https://fluent.github.io/helm-charts
"fluent" has been added to your repositories
```

Скачаем **values** **Fluent Bit** и сохраним как **fluentbit.values.yaml**.
```bash
wget -O fluentbit.values.yaml https://raw.githubusercontent.com/fluent/helm-charts/fluent-bit-0.10.0/charts/fluent-bit/values.yaml
```

Установим **Fluent Bit**.
```bash
helm upgrade --install fluent-bit fluent/fluent-bit --version 0.10.0 --namespace observability  -f fluentbit.values.yaml
```

# Мониторинг ElasticSearch

Доавим helm repo.
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
Установим чарт **Prometheus-Stack**

Скачаем **values** и сохраним как **prometheus-stack.values.yaml**.
```bash
wget -O prometheus-stack.yaml https://raw.githubusercontent.com/prometheus-community/helm-charts/kube-prometheus-stack-13.4.1/charts/kube-prometheus-stack/values.yaml
```
```console
helm upgrade --install prom-stack prometheus-community/kube-prometheus-stack \
--version 13.4.1 \
--namespace observability \
-f prometheus-stack.yaml
```

Учтановим чарт **Elasticsearch Exporter**, задав адрес **elasticsearch** и включив **serviceMonitor**.
```bash
helm upgrade --install elasticsearch-exporter prometheus-community/prometheus-elasticsearch-exporter \
--version 4.1.0 \
--namespace observability \
--set es.uri=http://elasticsearch-master:9200 \
--set serviceMonitor.enabled=true
```

Перейдем в графану http://grafana.34.90.137.122.xip.io/login

Добавим в grafana [Dashboard Elasticsearch exporter](https://grafana.com/grafana/dashboards/4358).

![elastic_dashboard](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-logging/elastic_dashboard.png)

Сделаем **drain** одной из нод **infra-pool** и убедимся, что метрики собираются.
```bash
kubectl drain gke-cluster-1-infra-pool-1e4e22a2-22gf --ignore-daemonsets --delete-emptydir-data
```
Статус Cluster Health остался зеленым, но количество нод в кластере уменьшилось до двух.

![nodes](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-logging/nodes.png)

Вернем ноду в работу.
```bash
kubectl uncordon gke-cluster-1-infra-pool-1e4e22a2-22gf
```
Добавим алерт на кол-во нод **Elasticsearch** в **prometheus-stack.yaml**
```yaml
additionalPrometheusRulesMap:
  elasticsearch-alertmanager.rules:
    groups:
      - name: elasticsearch
        rules:
          - alert: Elasticsearch_Too_Few_Nodes_Running
            expr: elasticsearch_cluster_health_number_of_nodes < 3
            for: 5m
            labels:
              severity: critical
            annotations:
              message: There are only {{$value}} < 3 ElasticSearch nodes running.
              summary: ElasticSearch running on less than 3 nodes

```
Обновим prometheus-stack
```console
helm upgrade --install prom-stack prometheus-community/kube-prometheus-stack \
--version 13.4.1 \
--namespace observability \
-f prometheus-stack.yaml
```
Алерт успешно был добавлен в prometheus.
![alert](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-logging/alert.png)

## Logs ingress-nginx
Логи nginx поступают к нам в виде строки.
```bash
log > 10.164.0.21 - - [31/Jan/2021:14:34:38 +0000] "POST /internal/search/ese/FmJJUkozUVpRU1VLNjRrNzZObm9QTXcdSGRKbEstWERUZ2E2Tjlsc2dob2VDUTo0NDIyMDM= HTTP/1.1" 200 45617 "http://kibana.34.90.137.122.xip.io/app/discover" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:78.0) Gecko/20100101 Firefox/78.0" 1390 0.145 [observability-kibana-kibana-5601] [] 10.80.3.5:5601 45545 0.145 200 d15e82f6cf34f5ad58c1e6259b0ebae5
```
Изменим формат логов генерируемые nginx.

Пропишем в nginx-ingress.values.yaml [log-format-escape-json](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#log-format-escape-json) и [log-format-upstream](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#log-format-upstream) .
```yaml
controller:
  config:
    log-format-escape-json: "true"
    log-format-upstream: '{"time": "$time_iso8601", "remote_addr": "$proxy_protocol_addr", "x_forward_for": "$proxy_add_x_forwarded_for", "request_id": "$req_id",
      "remote_user": "$remote_user", "bytes_sent": $bytes_sent, "request_time": $request_time, "status": $status, "vhost": "$host", "request_proto": "$server_protocol",
      "path": "$uri", "request_query": "$args", "request_length": $request_length, "duration": $request_time,"method": "$request_method", "http_referrer": "$http_referer",
      "http_user_agent": "$http_user_agent" }'
```
Обновим **nginx-ingress**.
```bash
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx --wait \
--namespace=nginx-ingress \
--create-namespace \
--version=3.22.0 \
-f nginx-ingress.values.yaml
```
Посмотрим логи в кибане.
### БЫЛО
![logs_1](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-logging/logs_1.png)

### СТАЛО
![logs_2](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-logging/logs_2.png)

# Loki

Доавим helm repo.
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
Установим чарт **Loki-Stack**

Скачаем **values** и сохраним как **loki-stack.values.yaml**.
```bash
wget -O loki-stack.yaml https://raw.githubusercontent.com/grafana/helm-charts/loki-stack-2.3.1/charts/loki-stack/values.yaml
```
```console
helm upgrade --install loki grafana/loki-stack --version 2.3.1 \
--namespace observability \
-f loki-stack.yaml
```
Добавим в **prometheus-stack.yaml** LOKI, указав его как источни данных для графаны.

```yaml
grafana:
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Loki
          type: loki
          access: proxy
          url: http://loki:3100
          jsonData:
            maxLines: 1000
```
Обновим prometheus-stack
```console
helm upgrade --install prom-stack prometheus-community/kube-prometheus-stack \
--version 13.4.1 \
--namespace observability \
-f prometheus-stack.yaml
```
### Создадим Dashboard, на котором одновременно выведем метрики nginx-ingress и его логи.

Включим **serviceMonitor** для **nginx-ingress**.

Отредактируем файл добавив в него.
```yaml
  metrics:
    enabled: true

    serviceMonitor:
      enabled: true
      namespaceSelector:
        any: true #Разрешим поиск SM из любого NS
```
Обновим  **nginx-ingress**.
```bash
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx --wait \
--namespace=nginx-ingress \
--create-namespace \
--version=3.22.0 \
-f nginx-ingress.values.yaml
```
Убедимся, что serviceMonitor стал доступ в **prometheus**.

![sm_ingress](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-logging/sm_ingress.png)

Создадим  **Dashboard** в файле **nginx-ingress.json**
