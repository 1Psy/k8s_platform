# Подготовка.
Установим следующее ПО:
```bash
kubectl https://kubernetes.io/docs/tasks/tools/install-kubectl/ 
kubectl-autocomplete https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete
minikube https://kubernetes.io/docs/tasks/tools/install-minikube/
k9s https://github.com/derailed/k9s
Kube Forwarder https://kube-forwarder.pixelpoint.io/
docker https://docs.docker.com/get-docker/
```

# Запуск minikube.
Запустим minikube с использованием docker driver.
```bash
minikube start --driver=docker
```
![minikube_start](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_intro/minikube_start.png)


```bash
kubectl cluster-info #Проверка подключения к кластеру.
minikube ssh #Зайти на контейнер с миникубом.
docker ps #Список всех запущенных контейнеров.
docker rm -f $(docker ps -a -q) #Удалим все контейнеры и дождемся их восстановления.
kubectl get pods -n kube-system # Все системные компоненты k8s в виде подов.
kubectl delete pod --all -n kube-system #Удалить все системные компоненты k8s и дождаться их восстановления.
kubectl get cs # Проверка состояния системных компонентов кластера.
```

## Задание.
Почему все pod в namespace kube-system восстановились после удаления.
Поды запущены как [статик поды]: https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/ и управляются управляются напрямую kubelet.

![static_pods](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_intro/static_pods.png)

Под coredns запущен как deployment и при удалении запускается снова.
![coredns_deployment](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_intro/coredns_deployment.png)


## Создать Dockerfile:
Для выполнения домашней работы необходимо создать Dockerfile, в котором будет описан образ:
1. Запускающий web-сервер на порту 8000 (можно использовать любой способ)
2. Отдающий содержимое директории /app внутри контейнера.
3. Работающий с UID 1001

## Сборка образа и пуш в registry.
```bash
docker build -t travk/web:0.1 .
docker push travk/web:0.1
```
## Создадим манифест web-pod.yaml с меткой app.
``` yaml
apiVersion: v1          # Версия API    
kind: Pod               # Объект, который создаем
metadata:              
  name: web             # Название Pod     
  labels:               # Метки в формате key: value
    name: app
spec:                   # Описание Pod
  containers:           # Описание контейнеров внутри Pod
  - name: web           # Название контейнера
    image: travk/web:0.1      # Образ из которого создается контейнер
```

```bash
kubectl apply -f web-pod.yaml #Применить созданный манифест.
kubectl get pods #Запросить список pods в текущем namespace.
kubectl get pod web -o yaml #Запрос манифеста из k8s со служебными данными.
kubectl describe pod web #Описание pod и его состояние.
```
## Добавим в наш манифест init контейнер генерирующий index.html.
Init-контейнеры -- это специальные контейнеры, которые запускаются при инициализации пода до запуска основных контейнеров.

``` yaml
apiVersion: v1 
kind: Pod     
metadata:    
  name: web    
  labels:
    name: app
spec:
  containers:
  - name: web
    image: travk/web:0.1
  initContainers:
  - name: init-busybox
    image: busybox:1.32.0
    commend: ['sh', '-c', 'wget -O- https://tinyurl.com/otus-k8s-intro | sh']
```
Commend init контейнера (аналог ENTRYPOINT в Dockerfile)

## Создадим volume и примаунтим его к контейнерам.
Для того, чтобы файлы, созданные в init контейнере, были доступны основному контейнеру в pod нам понадобится использовать volume типа emptyDir.

``` yaml
apiVersion: v1 
kind: Pod     
metadata:    
  name: web    
  labels:
    name: app
spec:
  containers:
  - name: web
    image: travk/web:0.1 
    volumeMounts:
    - name: app
      mountPath: /app
  initContainers:
  - name: init-busybox
    image: busybox:1.32.0
    command: ['sh', '-c', 'wget -O- https://tinyurl.com/otus-k8s-intro | sh'] #Command аналог ENTRYPOINT в Dockerfile.
    volumeMounts:
    - name: app
      mountPath: /app
  volumes:
  - name: app
    emptyDir: {}
```

## Как проверить работоспособность:

![run_pod_init](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_intro/run_pod_init.png)

```bash
kubectl delete pod web           #Удалим старый под.
kubectl apply -f web-pod.yaml    #Применяем новый манифест.
kubectl get pods -w              #Проследим за запуском пода.
kubectl port-forward --address 0.0.0.0 pod/web 8000:8000 #Запустим перенаправление портов из кластера на локальны ПК/
http://localhost:8000/index.html #Откроем url на локальном ПК и убедимся что сгенерированная init контейнером страница доступана.
```
В качестве альтернативы kubectl port-forward можно использовать удобную обертку [Kube-forwarder](https://kube-forwarderpixelpoint.io)


## Подготовим образ frontend из микросервис магазина Hipster Shop.
```bash
https://github.com/GoogleCloudPlatform/microservices-demo.git                #Склонируем репозиторий.
docker build -t travk/hipster-frontend:0.1 microservices-demo/src/frontend/  #Соберем сервис frontend.
docker push travk/hipster-frontend:0.1                                       #Запушим образ в docker hub.
```
## Создадим pod из командной строки.
```bash
kubectl run frontend --image travk/hipster-frontend:0.1 --restart=Never
kubectl run frontend --image travk/hipster-frontend:0.1 --restart=Never --dry-run=server -o yaml > frontend-pod.yaml

--dry-run #Must be "none", "server", or "client". If client strategy, only print the object that would be sent, without sending it. If server strategy, submit server-side request without persisting the resource.
-o yaml #Форматирование вывода в YAML
> frontend-pod.yaml #Перенаправление вывода в файл
```
## Выясните причину, по которой pod frontend находится в статусе Error
```bash
kubectl logs frontend #Проверим логи пода.
#Cервис не может запутиться т.к не заданы перменные.
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set
```
## Создадим новый манифест frontend-pod-healthy.yaml с добавленными env.
``` yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: frontend-healthy
  name: frontend-healthy
spec:
  containers:
  - image: travk/hipster-frontend:0.1
    name: frontend
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
  restartPolicy: Never
```

```bash
kubectl apply -f frontend-pod-healthy.yaml #Применим манифест
kubectl get pods                           #Убедимся, что под запущен.
```