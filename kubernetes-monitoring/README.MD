## Подготовка.
```bash
mkdir kubernetes-monitoring && cd kubernetes-monitoring
```

### Образ nginx.
Создадим кастомный образ nginx который будет:
* отдавать метрики
* использовать nginx-exporter для преобразования метрик в формат prometheus.

Возьмем набор файлов **nginx** из *kubernetes-intro/web* и добавим в конфиг **nginx** следующие строки для отдачи метрик.
```bash
    location = /basic_status {
        stub_status;
    }
```
Соберем образ и отправим в registry.
```bash
cd nginx/ #Переёдем в папку с Dockerfile и файлами для сборки.
docker build -t travk/nginx:0.1 .
docker push travk/nginx:0.1
```
Создадим **Deployment** файл и добавим в него **nginx/nginx-prometheus-exporter:0.8.0** который будет собирать метрики **nginx** и преобразовывать в формат **prometheus**.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: travk/nginx:0.1
          ports:
            - containerPort: 8000
        - name: nginx-exporter
          image: nginx/nginx-prometheus-exporter:0.8.0
          args:
            - -nginx.scrape-uri=http://localhost:8000/basic_status
          ports:
            - name: metrics
              containerPort: 9113 #Порт по которму exporter отдаёт метрики.
```
# Kube Prometheus Stack
* https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack

## Добавим репозитой.

```console
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
## Файл values.
Скачаем его.
```console
helm show values prometheus-community/kube-prometheus-stack --version 13.4.1 > values.yaml
```
Выключим поле, запрщающее использовать **servicemonitor** из других **ns**.
```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
```
## Установим Helm Chart

```console
helm upgrade --install prom-stack prometheus-community/kube-prometheus-stack \
--version 13.4.1 \
--namespace monitoring \
--create-namespace \
-f values.yaml
```
Наш чарт успешно разверут.

![kube-prom-stack](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-monitoring/kube-prom-stack.png)

Создадим **service**.
```yaml
kind: Service
apiVersion: v1
metadata:
  name: nginx-svc
  labels:
    app: nginx-svc
spec:
  selector:
    app: nginx
  ports:
    - name: metrics
      port: 9113
```
Создадим **servicemonitor**.
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-metrics
spec:
  selector:
    matchLabels:
      app: nginx-svc
  endpoints:
    - port: metrics
      path: /metrics
```
Применим их.
```bash
> kubectl apply -f service.yaml
service/nginx-svc created
> kubectl apply -f servicemonitor.yaml
servicemonitor.monitoring.coreos.com/nginx-metrics created
```

# Kube Prometheus Stack
### Пробросим prometheus на localhost.
```bash
kubectl port-forward svc/prom-stack-kube-prometheus-prometheus 9091:9090 -n monitoring
```
### Зайдем в **prometheus** http://localhost:9090/ и посмотрим **targets**.
![sm_targets](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-monitoring/sm_targets.png)

Enpoints наших pods доступны и метрики успешно поступают.

### Проверим, что метрики нашего приложения поступают.
![nginx_metrics](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-monitoring/nginx_metrics.png)

## Grafana
Пробросим порты и перейдем на http://localhost/ логин и пароль по-умолчанию  **admin prom-operator**.
```bash
> kubectl port-forward svc/prom-stack-grafana 80:80 -n monitoring
Forwarding from 127.0.0.1:80 -> 3000
Forwarding from [::1]:80 -> 3000
```
Добавим в графану дашбор для **nginx-exporter** https://grafana.com/grafana/dashboards/12708

![grafana_dashboard](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-monitoring/grafana_dashboard.png)

Видим, что метрики успешно поступают из **prometheus** и выстраиваются в виде графиков.