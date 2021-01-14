# Регистрация в Google Cloud Platform
 * Зарегистрируем аккаунт гугл и подтвердим номер телефона.
 * Перейдем на https://cloud.google.com/free/ и нажмем на активацию триала.
 * Выпустим виртуальную карту и добавим ее при активции триала.

# Google Cloud SDK
Инструкции для друх OS можно найти [тут.](https://cloud.google.com/sdk/docs/install#deb) 
## Установка
```bash
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list
sudo apt-get install apt-transport-https ca-certificates gnupg
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg
sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -
sudo apt-get update && sudo apt-get install google-cloud-sdk
```
Выполним инициализацию и авторизуемся под учеткой gcp.
```bash
 gcloud init 
 ```

# Создадим кластер GKE.
Включим службы.

```bash
gcloud services enable container.googleapis.com #Включим службу API Service.
gcloud services enable compute.googleapis.com   #Включим службу Compute Engine API.
```
Установим проект по-умолчанию.
```bash
gcloud projects list #Вывести список всех проектов.
gcloud config set project digital-waters-300922 #Установим проект по-умолчанию.
```
Установим регион по-умолчанию.
```bash
gcloud compute zones list #Получим список всех зон.При первом запросе, возможно потребуется активация Compute Engine API.
gcloud config set compute/zone europe-west4-b #Установим зону по-умолчанию.
```
Создадим кластер k8s.
```bash
gcloud container clusters create cluster-test1 --num-nodes=1
```
Добавим вторую ноду в кластер.
```bash
gcloud container node-pools list --cluster cluster-test1
gcloud container clusters resize cluster-test1 --node-pool default-pool --num-nodes 2
```

Получим учетные данные для кластера.
```bash
gcloud container clusters get-credentials cluster-test1
```
Проверим всё ли ок.
```bash
kubectl cluster-info
```
![k8s_cluster_gke](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/k8s_cluster_gke.png)

# HELM 3.
Установим helm скриптом.
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
Проверим версию.
```bash
helm version
```
![helm3](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/helm3.png)

Добавим **stable** репозиторий.
```bash
helm repo add stable https://charts.helm.sh/stable
```
Вывести список установленных репозиториев.
```bash
helm repo list
```
# nginx-ingress
Добавим helm репозиторий.
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
Установим **nginx-ingress**.
```bash
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx --wait \
--namespace=nginx-ingress \
--create-namespace \
--version=3.19.0
```
Используемые команды.

* **--wait** - ожидать успешного окончания установки
* **--timeout** - считать установку неуспешной по истечении указанного времени
* **--namespace** - установить chart в определенный namespace (еслине существует, необходимо создать)
* **--create-namespace** - создать неймспейс, если не существует.
* **--version** - установить определенную версию chart


# Cert-manager
## Установка cert-manager

Создадим namespace cert-manager:
```bash
$ kubectl create namespace cert-manager
```
Добавим Jetstack Helm репозиторий:

```bash
$ helm repo add jetstack https://charts.jetstack.io
```
Обновим кеш нашего локального Helm репозитория:

```bash
$ helm repo update
```

Установим `CustomResourceDefinition` ресурсы с помощью `kubectl`:

```bash
# Kubernetes 1.15+
$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.crds.yaml

```

Установим cert-manager Helm chart:

```bash
$ helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.1.0 \
```
## Проверим установку.

```bash
$ kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-56d6bbcb86-lmn4k              1/1     Running   0          28s
cert-manager-cainjector-6dd56cf757-7v5j4   1/1     Running   0          28s
cert-manager-webhook-6f84f655fb-mdthj      1/1     Running   0          28s
```
## Настройка Let's Encrypt ClusterIssuer
Т.к  Let's Encrypt  имеет на выдачу сертификатов, создадим **ClusterIssuer** **letsencrypt-staging** для тестов и **letsencrypt-prod** для рабочих решений.
```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-staging
   spec:
     acme:
       # The ACME server URL
       server: https://acme-staging-v02.api.letsencrypt.org/directory
       # Email address used for ACME registration
       email: user@example.com
       # Name of a secret used to store the ACME account private key
       privateKeySecretRef:
         name: letsencrypt-staging
       # Enable the HTTP-01 challenge provider
       solvers:
       - http01:
           ingress:
             class:  nginx
```
```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: letsencrypt-prod
   spec:
     acme:
       # The ACME server URL
       server: https://acme-v02.api.letsencrypt.org/directory
       # Email address used for ACME registration
       email: user@example.com
       # Name of a secret used to store the ACME account private key
       privateKeySecretRef:
         name: letsencrypt-prod
       # Enable the HTTP-01 challenge provider
       solvers:
       - http01:
           ingress:
             class: nginx
```

Применим манифесты.
```bash
kubectl apply -f kubernetes-templating/cert-manager
issuer.cert-manager.io/letsencrypt-prod created
issuer.cert-manager.io/letsencrypt-staging created
```
Проверим прошла ли регистрация на ACME.
```bash
kubectl describe issuer letsencrypt-staging
kubectl describe issuer letsencrypt-prod
```

![cert_manager_reg_status](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/cert_manager_reg_status.png)

# Chartmuseum.
## Установка Chartmuseum

Добавим репозиторий **chartmuseum**.
```bash
helm repo add chartmuseum https://chartmuseum.github.io/charts
helm repo update
```
Скачаем values чарта.
```bash
helm show values stable/chartmuseum > chartmuseum/values.yaml
```

Модифицируем его, реализовав там.
* Создание ingress ресурса с корректным hosts.name (должен использоваться nginx-ingress)
* Автоматическую генерацию Let's Encrypt сертификата

Установим 
```bash
helm upgrade --install chartmuseum chartmuseum/chartmuseum --wait \
--namespace=chartmuseum \
--create-namespace \
--version=2.15.0 \
-f chartmuseum/values.yaml
```

Сервис развернулся и был выдан ssl сертификат.

![ChartMuseum](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/ChartMuseum.png)

## Работа с ChartMuseum
Создадим тестовый чарт.
```bash
helm create test-chart
```
Упакуем чарт, версия берется из файла Chart.yaml
```bash
helm package test-chart
```
Загрузим наш пакет **test-chart-0.1.0.tgz**
```bash
curl --data-binary "@test-chart-0.1.0.tgz" https://chartmuseum.34.91.159.205.xip.io/api/charts
```
Добавим репозиторий **my-chartmuseum**
```bash
helm repo add my-chartmuseum https://chartmuseum.34.91.159.205.xip.io
```
Обновим инфорфмацию о добавленном чарте.
```bash
helm repo update
```
Выполнив поиск, мы можем увидеть наш чарт в удаленном репозитории.
```bash
> helm search repo test-chart
NAME                            CHART VERSION   APP VERSION     DESCRIPTION
my-chartmuseum/test-chart       0.1.0           1.16.0          A Helm chart for Kubernetes
```
Установим чарт.
```bash
helm install test-chart my-chartmuseum/test-chart --version=0.1.0
```
Чарт успешно установлен и наш тестовый сервис доступен.

![test-chart](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/test-chart.png)

# Harbor
## Установка Harbor

Добавим репозиторий **Harbor**.
```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
```
Скачаем values чарта.
```bash
helm show values harbor/harbor > harbor/values.yaml
```

Модифицируем его, реализовав там.
* Создание ingress ресурса с корректным hosts.name (должен использоваться nginx-ingress)
* Автоматическую генерацию Let's Encrypt сертификата

Установим 
```bash
helm upgrade --install harbor harbor/harbor \
--namespace=harbor \
--create-namespace \
--version=1.5.2 \
-f harbor/values.yaml
```
Сервис развернулся и был выдан ssl сертификат.

![harbor](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/harbor.png)

# Helmfile

## Установка helmfile.
```bash
wget https://github.com/roboll/helmfile/releases/download/v0.136.0/helmfile_linux_amd64
chmod +x ./helmfile_linux_amd64
sudo mv ./helmfile_linux_amd64 /usr/local/bin/helmfile
 ```
### Установка плагина helm diff.
```bash
helm plugin install https://github.com/databus23/helm-diff
```

Создадим **helmfile** с сервисами **nginx-ingress**, **cert-manager** и **harbor**.

```yaml
helmDefaults:
  wait: true
  timeout: 600
  # when using helm 3.2+, automatically create release namespaces if they do not exist (default true)
  createNamespace: true
#  verify: true
repositories:
  - name: ingress-nginx
    url: https://kubernetes.github.io/ingress-nginx
  - name: jetstack
    url: https://charts.jetstack.io
  - name: harbor
    url: https://helm.goharbor.io

releases:
  - name: nginx-ingress
    namespace: nginx-ingress
    chart: ingress-nginx/ingress-nginx
    version: 3.19.0

  - name: cert-manager
    namespace: cert-manager
    chart: jetstack/cert-manager
    version: v1.1.0
    hooks:
      - events: ["presync"]
        showlogs: true
        command: "kubectl"
        args:
          [
            "apply",
            "-f",
            "https://github.com/jetstack/cert-manager/releases/download/v1.1.0/cert-manager.crds.yaml",
          ]
      - events: ["postsync"]
        showlogs: true
        command: "kubectl"
        args: ["apply", "-f", "../cert-manager"]

  - name: harbor
    namespace: harbor
    chart: harbor/harbor
    version: 1.5.2
    values:
      - "../harbor/values.yaml"

```

Применим **helmfile**.
```bash
helmfile -f helmfile/helmfile.yaml apply
```
Сервисы были успешной развернуты.

# Создади helm chart приложения hipster-shop.

Инициализируем создание helm chart.
```bash
helm create kubernetes-templating/hipster-shop
```
Очистим папку **templates** и файл **values.yaml** и добавим в папку **templates** файл all-hipster-shop.yaml со следующим [содержимым](https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/v0.1.5/release/kubernetes-manifests.yaml).

Наш простой **helm chart** готов, развернём его.
```bash
helm upgrade --install hipster-shop kubernetes-templating/hipster-shop \
--namespace=hipster-shop \
--create-namespace 
```
Пробросим порт и убедимся, что сервисы развернулись.
```bash
kubectl get pods
kubectl port-forward service/frontend 80:80
```

![hipster-shop](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/hipster-shop.png)

# Создади helm chart для микросервиса frontend.

Создадим заготовку чарта и очистим ее.
```bash
helm create kubernetes-templating/frontend
rm -rf kubernetes-templating/frontend/templates/*
:> kubernetes-templating/frontend/values.yaml
```
Перенесём манифесты **frontend** из файла [**all-hipster-shop.yaml**](https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/v0.1.5/release/kubernetes-manifests.yaml) в папку **templates**.

```bash
deployment.yaml #Перенесем из файла.
service.yaml    #Перенесем из файла.
ingress.yaml    #Создадим самостоятельно.
```
Удалим из **kubernetes-templating/hipster-shop/templates/all-hipster-shop.yaml** строки **deployment** и **service** для
**frontend** и переустановим чарт без с обновленным шаблоном.
```bash
helm upgrade --install hipster-shop kubernetes-templating/hipster-shop \
--namespace=hipster-shop \
--create-namespace 
```
Проверим, что **deployment** и **service** для **frontend** удалились.
```bash
kubectl get all | grep frontend
```
Развернём чарт **frontend**.
```bash
helm upgrade --install frontend kubernetes-templating/frontend \
--namespace=hipster-shop \
--create-namespace 
```
Проверим, что чарт применился и ресурсы были созданы.
```bash
kubectl get all | grep frontend
pod/frontend-7ccb56cff9-49qtz             
service/frontend              
deployment.apps/frontend                
replicaset.apps/frontend-7ccb56cff9
```

![frontend](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/frontend.png)

Сервис успешно развернулся, так же мы получили через cert manager ssl сертификат.

## Шаблонизация frontend.
Вынесем в **values.yaml** переменную с 
 * **docker tag**
 * **Port**
 * **targetPort**
 * **NodePort**
 * Опционально - тип сервиса.
 * Ключ **NodePort** должен появиться в манифесте только если тип сервиса - **NodePort**
 * Другие параметры

Удалим старый релиз **frontend**.
```bash
helm delete frontend -n hipster-shop
```
Добавим **chart** **frontend** как зависимость в **hipster-shop**.Для этого в **Chart.yaml** добавим следющее.
```yaml
dependencies:
- name: frontend
  version: "1.0.1"
  repository: "file://../frontend"
```

Обновим зависимости, после выполнения команды, будет скачан архив **frontend-0.1.0.tgz** в папку **charts**.
```bash
helm dep update kubernetes-templating/hipster-shop
```
Обновим релиз hipster-shop и убедимся, что зависимость frontend установилась.
```bash
helm upgrade --install hipster-shop kubernetes-templating/hipster-shop \
--namespace=hipster-shop \
--create-namespace 
```
Чарт был обновлен и зависимость fronted была установлена.
```bash
kubectl get all | grep frontend
pod/frontend-7ccb56cff9-wqs95
service/frontend    
deployment.apps/frontend 
replicaset.apps/frontend-7ccb56cff9
```
## Helm-secrets

Установим helm plugin для шифрования секретов.
```bash
helm plugin install https://github.com/jkroepke/helm-secrets
```
Сгенерируем PGP ключ.
```bash
gpg --full-generate-key
```

Проверим его.
```bash
> gpg -k
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
/root/.gnupg/pubring.kbx
------------------------
pub   rsa4096 2021-01-13 [SC]
      7747655D179606F738EC134A540C6568FDAAE8B0
uid           [ultimate] Sergey <ololo@oyaebu.ru>
sub   rsa4096 2021-01-13 [E]
```

Создадим файл **secrets.yaml** в папке **frontend**.
Со следующим содержанием.
```yaml
visibleKey: hiddenValue
```

Зашифруем его.
```bash
sops -e -i --pgp 47E52982AC99C11A933CC2044B0975C500A2CCFF secrets.yaml
```
Для WSL.
```bash
export GPG_TTY=$(tty) 
```

Посмотрим на его зашифрованное содержание.
```yaml
> cat secrets.yaml
visibleKey: ENC[AES256_GCM,data:3vXSAm05+9ilh2Q=,iv:RMyN08iCkl/TdV1QJWJpu6qyTakwvvQR8MrAA1LhqMY=,tag:Z92NxrbECOInN4Uepx9pDg==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    lastmodified: '2021-01-13T17:28:39Z'
    mac: ENC[AES256_GCM,data:jM96rqbG8wT1A+I4z1lPOIUPA3av6RX4beaIQk2KQx4p/ckSMWFZ1/QcJHhtjKjmwOifl/x6dT+dE78EQG+bDvxnslrDFazRtdKGm6lo+L3hdfDI4H3MSQB2xsDLMF5OHTwtcbT+hqDXHNo2gmlajYx4IKtlOoktp1EUAlgyqrY=,iv:k/tbDL07/v2pCkxASEXe4JD4q7/JQVK/aAX+YXk0uY0=,tag:n+Ccpm3AF4vckqNiyk1J4A==,type:str]
    pgp:
    -   created_at: '2021-01-13T17:28:24Z'
        enc: |
            -----BEGIN PGP MESSAGE-----

            hQGMA8I6k+TKTzraAQv8DyKgkwt+5zeqe3PnUgVld7D3H5STh856e68G/rvwcsF3
            UGarQgyL3K56jstXCPm6jfgLgJYkn7aCHGmrQIqx+yD4IUfVAhm7xdC5mun7np64
            Tpu3fQuHs20WEn+hc0ATkz151sr6Qhb1wmudO5Q6I8toJZnOpYQvXKCWEF/uBfBd
            jE19MzKUC/Eyhpm+hVXLiu31JEumaI2UorfWOlwJY7ULlEoKuzX1iQg0xOb/23rq
            MSwCyIk6uFVKSRYH9cRVkbZVgeUJD3mi2vkCNRok4nCEgIUYTVwAI7ns4lPjJ71O
            uHULLK59yXKtkElr40DmXQm9ucszaYYsdCDJSVVR8LhUaAUMtRjXMf0aSHg3HrvZ
            k2BLS70qQ0T1d9MPssG89VatkqjF/D5fWZb+w71qVC3uGJWtVC7egf55p7l+SsUY
            Iq5Tz+paEfUVTjDCe+wvfMIkAv1j8/1TCck/gS6AF5KGJwk7fXZyb86QhjD4J1Hj
            hqJUJBG7vh0cT4ock9iY0l4BWR45VktAGJApOXL4RbVH/n6+ROG4XxY9VJDaPBuK
            W8DjrFFqqDShhlYLrZEIDdA1iojI+Cy9jnh1exxF8EkFjqpuSmPDZPcUpru711JC
            3wI+BZuwEtlfKDheQlNH
            =LPs4
            -----END PGP MESSAGE-----
        fp: 47E52982AC99C11A933CC2044B0975C500A2CCFF
    unencrypted_suffix: _unencrypted
    version: 3.6.1
```

Расшифовать секрет можно так.
```bash
# helm secrets
helm secrets view secrets.yaml
# sops
sops -d secrets.yaml
```

Создадим файл **secret_temp.yaml** в папке **frontend/templates**.
Со следующим содержанием.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret
type: Opaque
data:
  visibleKey: { { .Values.visibleKey | b64enc | quote } }
```

Передадим файл **secrets.yaml** как **values.yaml**

Плагин **helm-secrets** расшифрует переменные **secrets.yaml** и передав значение в шаблон **secret_temp.yaml**

Во время установки чарта будет создан временный файл **secrets.yaml.dec**
```bash
helm secrets upgrade --install frontend kubernetes-templating/frontend --namespace hipster-shop \
-f kubernetes-templating/frontend/values.yaml \
-f kubernetes-templating/frontend/secrets.yaml
```
Проверим, что наш секрет был расшифрован, подставлен в шаблон и применен.
```bash
kubectl get secrets
```
![helm_secret](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/helm_secret.png)

# Kubecfg

Из манифестов  микросервисов **paymentservice** и **shippingserviceс** (файл **kubernetes-templating/hipster-shop/templates/all-hipstershop.yaml**) 

скопируем **service** и **deployment** в директорию **kubernetes-templating/kubecfg**.

Получим следующие файлы.
```bash
paymentservice-deployment.yaml
paymentservice-service.yaml
shippingservice-deployment.yaml
shippingservice-service.yaml
```

Обновим hipster-shop
```bash
helm upgrade --install hipster-shop kubernetes-templating/hipster-shop \
--namespace=hipster-shop \
--create-namespace 
```
Релиз успещно обновился и два сервиса пропали.

![kube_pods](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-templating/kube_pods.png)
