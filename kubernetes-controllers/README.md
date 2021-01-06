# Подготовка
## Установим kind.
https://kind.sigs.k8s.io/docs/user/quick-start/#installation
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.9.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /usr/local/bin/kind
```
![kind_install](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/kind_install.png)

## Запустим кластер kind.
Создадим конфигурационный файл кластера с 3 master и 3 worker нодами.
``` yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: control-plane
- role: control-plane
- role: worker
- role: worker
- role: worker
```
```bash
kind create cluster --config kind-config.yaml #Запустим создание кластера kind.
```
![kind_cluster_install](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/kind_cluster_install.png)

Убедимся, что кластер запущен и все ноды в статусе Ready.
```bash
kubectl cluster-info
kubectl get nodes
```
![kind_cluster_nodes](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/kind_cluster_nodes.png)

# ReplicaSet
## Запустим микросервисы frontend через ReplicaSet контроллер.
- Создадим манифест frontend-replicaset.yaml.
- Укажем selector.
- Добавим env, без них сервис не заработает.
``` yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend  
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: travk/hipster-frontend:0.1
        env:
        - name: PRODUCT_CATALOG_SERVICE_ADDR
          value: "null"
        - name: CURRENCY_SERVICE_ADDR
          value: "null"
        - name: CART_SERVICE_ADDR
          value: "null"
        - name: RECOMMENDATION_SERVICE_ADDR
          value: "null"
        - name: SHIPPING_SERVICE_ADDR
          value: "null"
        - name: CHECKOUT_SERVICE_ADDR
          value: "null"
        - name: AD_SERVICE_ADDR
          value: "null"
```          
```bash
kubectl apply -f frontend-replicaset.yaml      #Применим манифест.
kubectl get pods -l app=frontend               #Получим список подов с лейблом app=frontend.
kubectl scale replicaset frontend --replicas=3 #Увеличим колличество реклик до 3.
kubectl get rs frontend                        #Проверим, что ReplicaSet управляет 3 репликами и они готовы к работе.
kubectl delete pods -l app=frontend | kubectl get pods -l app=frontend -w #Удалим поды и посмотрим как они восстановятся.
```
![replicaset_frontend](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/replicaset_frontend.png)

```bash
kubectl apply -f frontend-replicaset.yaml #Повторно применим манифест frontend-replicaset.yaml.
```
![replicaset_frontend_one_pod](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/replicaset_frontend_one_pod.png)

Увеличим колличество реплик в нашем манифесте до 3.
``` yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 3
...
```    
```bash
kubectl apply -f frontend-replicaset.yaml      #Применим манифест.
kubectl get rs frontend                        #Проверим, что реплик стало 3.
```  
![replicaset_frontend_three_pod](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/replicaset_frontend_three_pod.png)

## Обновление ReplicaSet
```bash
docker tag travk/hipster-frontend:0.1 travk/hipster-frontend:0.2 #Сделаем retag образа на версию 0.2
docker push travk/hipster-frontend:0.2                   #Запушим образ с новым тегом в dockerhub.
```

Обновим образ в манифесте на новую версию.
```yaml
...
    spec:
      containers:
      - name: server
        image: travk/hipster-frontend:0.2
...
```
Примим манифест и посмотрим состояние запуска.
```bash
kubectl apply -f frontend-replicaset.yaml | kubectl get pods -l app=frontend && kubectl describe pod -l app=frontend | grep Image:
```
Поды не перезапустились и версия осталась старая.

Так как ReplicaSet контроллер следит за колличеством запущенных подов и не проверяет соттвествую ли запущенные поды шаблону.
![replicaset_frontend_pod](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/replicaset_frontend_pod.png)

Проверим это удалив поды.
```bash
kubectl delete pods -l app=frontend
kubectl describe pod -l app=frontend | grep Image:
```
Поды перезапустились с новым тегом образа.

![replicaset_frontend_pod_0.2](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/replicaset_frontend_pod_0.2.png)


# Deployment
## Запустим микросервисы paymentservice через ReplicaSet контроллер.
Подготовим образ paymentservice из микросервис магазина Hipster Shop.
```bash
https://github.com/GoogleCloudPlatform/microservices-demo.git                                #Склонируем репозиторий.
docker build -t travk/hipster-paymentservice:0.1 microservices-demo/src/paymentservice/      #Соберем сервис paymentService.
docker tag travk/hipster-paymentservice:0.1 travk/hipster-paymentservice:0.2                 #Сделаем retag образа на версию 0.2
docker push travk/hipster-paymentservice:0.1 && docker push travk/hipster-paymentservice:0.2 #Запушим образ в docker hub.
```
Создадим манифест paymentservice-replicaset.yaml с новым микросервисом c 3 репликами и тегом 0.1.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: paymentservice
  labels:
    app: paymentservice
spec:
  replicas: 3
  selector:
    matchLabels:
      app: paymentservice
  template:
    metadata:
      labels:
        app: paymentservice
    spec:
      containers:
      - name: paymentservice
        image: travk/hipster-paymentservice:0.1
```
Скопируем содержимое файла paymentservicereplicaset.yaml в файл paymentservice-deployment.yaml и заменим kind с **ReplicaSet** на **Deployment**

```bash
cp paymentservice-replicaset.yaml paymentservice-deployment.yaml

apiVersion: apps/v1
kind: Deployment
...
```
Применим манифест и убедимся , что запустились 3 реплики и они готовы.
Так же обратим внимание, что при создании deployment был созда replicaset.
```bash
kubectl apply -f paymentservice-deployment.yaml
kubectl get deployment
kubectl get rs
```
![deployment_paymentservice](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/deployment_paymentservice.png)


Поменяем версию в paymentservice-deployment.yaml на 0.2 и применим его.

```bash
kubectl apply -f paymentservice-deployment.yaml | kubectl get pods -l app=paymentservice  && kubectl describe pods -l app=paymentservice | grep Image:
```
Произошло обновление Rolling Update, был создан новый ReplicaSet с версией 0.2, который развернул 3 пода.
ReplicaSet с версией 0.1 уменьший колличество реплик до 0.

![deployment_paymentservice_pods](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/deployment_paymentservice_pods.png)

Посмотрим историю версий Deployment и произведем откат на одну ревизию назад.

```bash
kubectl rollout history deployment paymentservice
kubectl rollout undo deployment paymentservice --to-revision=5 | kubectl get rs -l app=paymentservice -w
```
![deployment_paymentservice_rollback](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/deployment_paymentservice_rollback.png)

# Стратегии развертывания.

## Реализуем 2 сценария развертывания.

Создадим 2 манифеста.
```bash
paymentservice-deployment-bg.yaml
paymentservice-deployment-reverse.yaml
```
* **Аналог blue-green:**
  1. Развертывание трех новых pod
  2. Удаление трех старых pod

**RollingUpdate** - Стратигия плавного обновления подов, конфигурируется с помощью [параметров](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)

```yaml
  strategy:
    type: RollingUpdate 
    rollingUpdate:
      maxSurge: 100%    #Колличество подов сверх желаемого кол-ва.
      maxUnavailable: 0 #Колличество подов недоступных в процессе обновления.
```
```bash
kubectl apply -f paymentservice-deployment-bg.yaml | kubectl get pods -l app=paymentservice-bg -w
kubectl describe deployment paymentservice-bg
```

![deployment_paymentservice_ru_gb](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/deployment_paymentservice_ru_gb.png)

Были созданы реплики с новой версией, только после этого, старые поды удалились.

* **Reverse Rolling Update:**
  1. Удаление одного старого pod
  2. Создание одного нового pod

```yaml
  strategy:
    type: Recreate #Стратегия при которой, перед развертыванием нового пода, старый должен быть удален.
```
```bash
kubectl apply -f paymentservice-deployment-reverse.yaml | kubectl get pods -l app=paymentservice-reverse -w
kubectl describe deployment paymentservice-reverse
```
![deployment_paymentservice_reverse](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/deployment_paymentservice_reverse.png)

# Probes

## [Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) - Механизм проверки работоспособности подов в k8s. 

Создадим манифест frontend-deployment.yaml с микросервисом frontend, 3 репликами и версий 0.1

Добавим проверку [readinessProbe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-readiness-probes)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontendservice
        image: travk/hipster-frontend:0.1
        readinessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/_healthz"
            port: 8080
            httpHeaders:
            - name: "Cookie"
              value: "shop_session-id=x-readiness-probe"
        env:
        - name: PRODUCT_CATALOG_SERVICE_ADDR
          value: "null"
        - name: CURRENCY_SERVICE_ADDR
          value: "null"
        - name: CART_SERVICE_ADDR
          value: "null"
        - name: RECOMMENDATION_SERVICE_ADDR
          value: "null"
        - name: SHIPPING_SERVICE_ADDR
          value: "null"
        - name: CHECKOUT_SERVICE_ADDR
          value: "null"
        - name: AD_SERVICE_ADDR
          value: "null"

```
Запустим деплоймент и убедимся, что проверки отработали.
```bash
kubectl apply -f frontend-deployment.yaml
kubectl get pods -l app=frontend 
```
Проверка завершилась успешно и под был запущен.

![probe_frontend_ok](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/probe_frontend_ok.png)

Посмотрим описание проверки и ее параметры.
```bash
kubectl describe pods -l app=frontend
```
![probe_frontend_ok_describe](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/probe_frontend_ok_describe.png)

Сымитируем некорректную работу приложения и посмотрим, как будет вести себя обновление:
  1. Заменим path: "/_healthz" на path: "/_health"
  2. Укажем версию 0.2
   
```yaml
...
        image: travk/hipster-frontend:0.2
        readinessProbe:
          initialDelaySeconds: 10
          httpGet:
            path: "/_health"
            port: 8080
            httpHeaders:
            - name: "Cookie"
...
```
Применим манифест и посмотрим статус.

```bash
kubectl apply -f frontend-deployment.yaml 
kubectl get pods -l app=frontend
```
![probe_frontend_neok](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/probe_frontend_neok.png)

Как мы видим, создался новый под, но не переходит в состояние READY.

Посмотрим описание пода.
```bash
kubectl describe pod frontend-586d55896b-qmtwg 
```
![probe_frontend_neok_describe](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/probe_frontend_neok_describe.png)

Пока readinessProbe для нового pod не станет успешной - Deployment сможет продолжить обновление.

Проверим успешность выполнения deployment с помощью команды.
```bash
kubectl rollout status deployment/frontend
```
Если проверка была провалена, получим следующее сообщение об ошибке.
![rollout_status_fail](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/rollout_status_fail.png)

Если проверка была успешно пройдена, мы увидим сообщения с прогрессом разворачивания.
![rollout_status_ok](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/rollout_status_ok.png)

Данная комнда может быть использована в CI/CD для получения статуса обвноления.

Ниже описан пример с шагами раавертывания и отката.
```yaml
deploy_job:
  stage: deploy
  script:
    - kubectl apply -f frontend-deployment.yaml
    - kubectl rollout status deployment/frontend --timeout=60s
rollback_deploy_job:
  stage: rollback
  script:
    - kubectl rollout undo deployment/frontend
  when: on_failure
  ```
# DaemonSet
При применении данного контроллера, на каждой ноде создается по одному Pod.

Создадим DaemonSet манифест node-exporter-daemonset.yaml с контейнером [Node_Exporter](https://github.com/prometheus/node_exporter)

Так же добавим tolerations для разворачивании на мастер нодах.

Посмотреть текущие taint ключи можно командой.
```bash
kubectl describe nodes | egrep "Name:|Taints:"
```
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.0.1
      tolerations:
        - key: "node-role.kubernetes.io/master"
          effect: "NoSchedule"
          operator: "Exists"
```
Применим его и проверим его.

```bash
 kubectl apply -f node-exporter-daemonset.yaml
 kubectl get pods -l app=node-exporter 
 kubectl get pods -o wide -l app=node-exporter 
 ```
 ![daemonset_all_nodes](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/daemonset_all_nodes.png)

Поды развернулись на всех нодах, включая мастер.

Запустим перенаправление портов на один из подов.
```bash
kubectl port-forward node-exporter-42ldl 9100:9100
```
Зайдем на локальном ПК http://localhost:9100/metrics и посмотрим на метрики с ноды где находится под.
 ![metrics_node](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/metrics_node.png)

