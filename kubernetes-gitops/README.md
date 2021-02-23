## Подготовка.
```bash
mkdir kubernetes-gitops && cd kubernetes-gitops
```
* Создадим аккаунт https://gitlab.com
* Создадим *Public project* **microservices-demo**

Переместим проект microservices-demo код из GitHub [репозитория](https://github.com/GoogleCloudPlatform/microservices-demo).

```
git clone https://github.com/GoogleCloudPlatform/microservices-demo
cd microservices-demo
git remote remove origin
git remote add origin git@gitlab.com:1Psy/microservices-demo.git
git push origin master
```

## Создадим helm чарты сервисов.

За основу возьмем yaml файл всех сервисов **realease/kubernetes-manifests.yaml** и вынесем в перменные следующие значения.
```bash
image:
repository: microservice-name
tag: latest
```
Взять готовые можно [тут](https://gitlab.com/1Psy/microservices-demo/-/tree/master/deploy/charts).

## Подготовка Kubernetes кластера.

Развернем 4 ноды типа **n1-standard-2** в **GCP**.
```bash
gcloud container clusters create cluster-gitop \
--machine-type "n1-standard-2" \
--num-nodes "4"
```
## Continuous Integration

Соберём образы для всех микросервисов с версией **v.0.0.1**.

Для этого воспользуемся скриптом *hack/make-docker-images.sh*.

Который соберет микросервисы и отправит их в наш докер хаб.
```bash
export TAG=v0.0.1 
export REPO_PREFIX=travk
./hack/make-docker-images.sh
```
Заменим образ в чартах на наш.
```bash
sed -i 's/v0.2.2/v0.0.1/g' deploy/charts/*/values.yaml
sed -i 's+gcr.io/google-samples/microservices-demo+travk+g' deploy/charts/*/values.yaml
```
# GitOps
## Установим FLux+Helm Operator

Add the Flux repository:
```bash
helm repo add fluxcd https://charts.fluxcd.io
```

Apply the Helm Release CRD:
```bash
kubectl apply -f https://raw.githubusercontent.com/fluxcd/helm-operator/master/deploy/crds.yaml
```
Create the flux namespace:
```bash
kubectl create ns flux
```

Create **flux.values.yaml**.
```bash
cat > flux.values.yaml <<EOF
git:
  url: git@gitlab.com:1Psy/microservices-demo.git
  path: deploy
  ciSkip: true	
  pollInterval: 1m
registry:
  automationInterval: 1m
EOF
```

Install Flux chart.
```bash
helm upgrade --install flux fluxcd/flux -f flux.values.yaml --namespace flux
```

Create **helm-operator.values.yaml**.
```bash
cat > helm-operator.values.yaml <<EOF
helm:
  versions: v3
git:
  pollInterval: 1m
  ssh:
    secretName: flux-git-deploy

chartsSyncInterval: 1m

logReleaseDiffs:
  true

configureRepositories:
  enable: true
  repositories:
    - name: stable
      url: https://charts.helm.sh/stable
EOF
```

Install Helm Operator.
```bash
helm upgrade --install helm-operator fluxcd/helm-operator -f helm-operator.values.yaml --namespace flux
```
### Fluxctl
* https://docs.fluxcd.io/en/stable/references/fluxctl/

```bash
wget https://github.com/fluxcd/flux/releases/download/1.21.2/fluxctl_linux_amd64
chmod +x ./fluxctl_linux_amd64
sudo mv ./fluxctl_linux_amd64 /usr/local/bin/fluxctl
 ```

### Настройка.

Получим публичный ключ для работы с репозитоием.
```bash
fluxctl identity --k8s-fwd-ns flux
```

![flux_pub_key](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-gitops/flux_pub_key.png)


Добавим ключ для учетной записи **Gitlab**.(Только для тестов.)

![add_key_gitlab](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-gitops/add_key_gitlab.png)

### Проверка

Добавим в директорию **deploy/namespaces** следующий манифест и закпушим в репозиторий.

```bash
cat > microservices-demo.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: microservices-demo
EOF
```
Т.к ранее мы настроили синхронизацию с папкой deploy, то в результате добавления нового манифеста будет создан ns.

![ns](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-gitops/ns.png)

## HelmRelease

Создадим папку **deploy/releases**.
```bash
mkdir deploy/releases && cd 
```

Поместим туда файл **frontend.yaml** с описанием конфигурации релиза.

```yaml
apiVersion: helm.fluxcd.io/v1
kind: HelmRelease
metadata:
  name: frontend
  namespace: microservices-demo
  annotations:
    fluxcd.io/ignore: "false"
    fluxcd.io/automated: "true"                      # Аннотация разрешает автоматическое обновление релиза в Kubernetes кластере в случае изменения версии Docker образа в Registry.
    flux.weave.works/tag.chart-image: semver:~0.0    # Указываем Flux следить за обновлениями конкретных Docker образов в Registry.
                                                     # Новыми считаются только образы, имеющие версию выше текущей и отвечающие маске семантического версионирования ~0.0 (например, 0.0.1, 0.0.72, но не 1.0.0.
spec:
  releaseName: frontend
  helmVersion: v3
  chart:                                             # Helm chart, используемый для развертывания релиза. В нашем случае указываем git-репозиторий, и директорию с чартом внутри него.
    git: git@gitlab.com:1Psy/microservices-demo.git
    ref: master
    path: deploy/charts/frontend
  values:                                            # Переопределяем переменные Helm chart. В дальнейшем Flux может сам переписывать эти значения и делать commit в git-репозиторий (например, изменять тег Docker образа при его обновлении в Registry).
    image:                                           # https://docs.fluxcd.io/en/latest/references/helm-operator-integration/
      repository: travk/frontend
      tag: 0.0.1
```
Запушим.

Проверим, что HelmRelease Frontend был успешно развёрнут.

![hr_frontend](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-gitops/hr_frontend.png)

Так же можно посмотреть статус helm с chart.
```bash
> helm list -n microservices-demo
NAME     NAMESPACE          REVISION UPDATED    STATUS   CHART           APP VERSION
frontend microservices-demo 2        2021-02-23 deployed frontend-0.1.0  0.2.2
```

Следующей командой, можно инициировать запуск сихронизации **flux** c репозиторием.
```bash
fluxctl --k8s-fwd-ns flux sync
```

### Обновим образ.

Сделаем ретег образра из **v0.0.1** в **v0.0.2** и запушим.
```bash
docker tag travk/frontend:v0.0.1 travk/frontend:v0.0.2
docker push travk/frontend:v0.0.2
```

Можем заметить, что выкатился новый релиз.
```bash
> helm list -n microservices-demo
NAME     NAMESPACE          REVISION UPDATED    STATUS   CHART           APP VERSION
frontend microservices-demo 3        2021-02-23 deployed frontend-0.1.0  0.2.2
```
```bash
> kubectl describe pod frontend-5ff8cbf598-h9wlx | grep Image:
    Image:          travk/frontend:v0.0.2
```
Так же был сделан коммит в репозиторий, который внёс изменения в **Helm Release frontend.yaml**.

![set_new_tag](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-gitops/set_new_tag.png)

### Обновление Helm chart.

Изменим **Helm Chart Frontend**, поменяем имя **deployment** на **frontend-hipster**

```bash
sed -i 's/name: frontend/name: frontend-boutique/g' deploy/charts/frontend/templates/deployment.yaml
```

Запушим изменения.

Посмотрим логи **Helm Operator**.
```log
checking 3 resources for changes targetNamespace=microservices-demo release=frontend
Created a new Deployment called "frontend-hipster" in microservices-demo targetNamespace=microservices-demo release=frontend 
updating status for upgraded release for frontend targetNamespace=microservices-demo release=frontend
```

**Helm Operator** произвел сихронизацию с репозиторием, заметил изменения в **deploymenet** , создал новый **helm** релиз, удалив старый.

* Добавим HelmRelease манифесты для остальных микросервисов, поместим их в папку **deploy/charts/releases/**.

Запросим всех Helmrelease.

![mcs_release](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-gitops/mcs_release.png)

Всё успешно развернулось.

## Полезные команды fluxctl.

* **export FLUX_FORWARD_NAMESPACE=flux** переменная окружения, указывающая на namespace, в который установлен flux (альтернатива ключу --k8s-fwd-ns <flux installation ns>)
* fluxctl list-workloads -a посмотреть все workloads, которые находятся в зоне видимости flux
* fluxctl list-images -n microservices-demo - посмотреть все Docker образы, используемые в кластере (в namespace microservices-demo)
* fluxctl automate/deautomate - включить/выключить автоматизацию управления workload

# Canary deployments с Flagger и Istio

## Установка Istio.

Установим Istio в кластер с помощью **istioctl**.
* https://istio.io/latest/docs/setup/getting-started/

Установим **istioctl**.
```bash
wget https://github.com/istio/istio/releases/download/1.9.0/istioctl-1.9.0-linux-amd64.tar.gz
tar xzvf istioctl-1.9.0-linux-amd64.tar.gz
chmod +x ./istioctl
sudo mv ./istioctl /usr/local/bin/istioctl
 ```
Установим **istio** с профилем **demo**.
 ```bash
 istioctl manifest apply --set profile=default.
 ```
## Установка Flagger.

```bash
helm repo add flagger https://flagger.app
kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml
```

Установим **flagger**, укажем использовать для работы **istio**.
```bash
helm upgrade --install flagger flagger/flagger \
    --namespace=istio-system \
    --set crd.create=false \
    --set meshProvider=istio \
    --set metricsServer=http://prometheus:9090
```

Добавим **sidecar pod istio** с **envoy proxy** во все сервисы в **ns** **microservices-demo**.

Для того добавил лейб инжектора истио в файл **deploy/namespaces/microservices-demo.yaml**.
```bash
cat > deploy/namespaces/microservices-demo.yaml <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: microservices-demo
  labels:
    istio-injection: enabled
EOF
```
Запушим и проверим синхронизацию.
```bash
> kubectl get ns microservices-demo --show-labels
NAME                 STATUS   AGE    LABELS
microservices-demo   Active   146m   fluxcd.io/sync-gc-mark=sha256.fOxXjb9cwEooYcBfadF2zr--YOTSJbgV_sAhfr-hDyw,istio-injection=enabled
```
Пересоздадим поды для добавления **sidecar**.
```bash
kubectl delete pods --all -n microservices-demo
```

Проверим, что появился sidecar istio-proxy.
```bash
kubectl describe pod -l app=frontend -n microservices-demo
```

Istio в качестве альтернативы классическому **ingress** предлагает свой набор абстракций.

Чтобы настроить маршрутизацию трафика к приложению с использованием **Istio**, нам необходимо добавить ресурсы [VirtualService](https://istio.io/docs/concepts/traffic-management/#virtual-services) и [Gateway](https://istio.io/docs/concepts/traffic-management/#gateways).

Они уже описаны в файле **istio-manifests/frontend-gateway.yaml**.

Применим их.
```bash
> kubectl apply -f istio-manifests/frontend-gateway.yaml
gateway.networking.istio.io/frontend-gateway created
virtualservice.networking.istio.io/frontend-ingress created
```

Проверим, что **gateway** создался.
```bash
> kubectl get gateway -n microservices-demo
NAME               AGE
frontend-gateway   34s
```

Узнаем внешний ip **istio-ingressgateway**. 
```
> kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                                                                      AGE
istio-ingressgateway   LoadBalancer   10.83.245.72   34.90.109.157   15021:32095/TCP,80:30251/TCP,443:30739/TCP,31400:30009/TCP,15443:30757/TCP   43m
```

Проверим, что наш сервис frontend доступен по внешнему ip.
![shop](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-gitops/shop.png)

Перенесем ресурсы **istio-manifests/frontend-gateway.yaml** в **helm chart** **frontend**.
```bash
cat > deploy/charts/frontend/templates/frontend-gateway.yaml <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: frontend-gateway
  namespace: microservices-demo
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "*"
EOF
```
```bash
cat > deploy/charts/frontend/templates/frontend-ingress-vs.yaml <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: frontend-ingress
  namespace: microservices-demo
spec:
  hosts:
    - "*"
  gateways:
    - frontend-gateway
  http:
    - route:
        - destination:
            host: frontend
            port:
              number: 80
EOF
```
Запушим.

## Flagger | Canary
* https://docs.flagger.app/usage/how-it-works#canary-custom-resource

Создадим файл **deploy/charts/frontend/templates/canary.yaml** для микросервиса **frontend**.

В нем будем хранить описание стратегии, по которой необходимо обновлять данный микросервис.

```bash
cat > deploy/charts/frontend/templates/canary.yaml <<EOF
apiVersion: flagger.app/v1beta1
kind: Canary
metadata:
  name: frontend
  namespace: microservices-demo
spec:
  provider: istio
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  progressDeadlineSeconds: 60
  service:
    port: 80
    targetPort: 8080
    gateways:
    - frontend-gateway
    hosts:
    - "*"
    trafficPolicy:
      tls:
        mode: DISABLE
  analysis:
    interval: 30s
    threshold: 5
    maxWeight: 30
    stepWeight: 5
    metrics:
    - name: request-success-rate
      threshold: 99
      interval: 30s
    - name: latency
      templateRef:
        name: latency
        namespace: istio-system
      thresholdRange:
        max: 500
      interval: 30s
EOF
```
Запушим и дождёмся создания canary ресурса.
```bash
kubectl get canary -n microservices-demo
```
Проверим, что к поду добавился префикс **primary**.
```bash
> kubectl get pods -n microservices-demo -l app=frontend-primary
NAME                                READY   STATUS    RESTARTS   AGE
frontend-primary-64d5b9c6f4-26d9n   2/2     Running   0          21s
```
Создадим версию v0.0.3
```bash
docker tag travk/frontend:v0.0.1 travk/frontend:v0.0.3
docker push travk/frontend:v0.0.3
```
Дождмся пока flux найдёт новый образ и выкатит его по канарейке.
```bash
kubectl describe canary frontend -n microservices-demo
```
```bash
  Normal   Synced  10m              flagger  New revision detected! Scaling up frontend.microservices-demo
  Normal   Synced  10m              flagger  Starting canary analysis for frontend.microservices-demo
  Normal   Synced  10m              flagger  Advance frontend.microservices-demo canary weight 5
  Normal   Synced  9m30s            flagger  Advance frontend.microservices-demo canary weight 10
  Normal   Synced  9m               flagger  Advance frontend.microservices-demo canary weight 15
  Normal   Synced  8m30s            flagger  Advance frontend.microservices-demo canary weight 20
  Normal   Synced  8m               flagger  Advance frontend.microservices-demo canary weight 25
  Normal   Synced  7m30s            flagger  Advance frontend.microservices-demo canary weight 30
  Normal   Synced  6m (x3 over 7m)  flagger  (combined from similar events): Promotion completed! Scaling down frontend.microservices-demo
```
