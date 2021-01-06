# Запуск minikube.

### Bin package
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
### Запустим minikube.
```bash
minikube start --driver=docker
minikube start --cpus=4 --memory=8192mb --vm-driver hyperv --hyperv-virtual-switch "Lan"
```
# INFO
#### **Kubelet** использует **liveness** пробу для проверки, когда перезапустить контейнер. Например, **liveness** проба должна поймать блокировку, когда приложение запущено, но не может ничего сделать. В этом случае перезапуск приложения может помочь сделать приложение более доступным, несмотря на баги.

#### **Kubelet** использует **readiness** пробы, чтобы узнать, готов ли контейнер принимать траффик. **Pod** считается готовым, когда все его контейнеры готовы.

# Добавление проверок Pod
### Откроем файл из предыдущего ДЗ **kubernetes-intro/web-pod.yml**
### Добавим проверку **readinessProbe**
```yaml
...
spec:                     # Описание Pod
  containers:             # Описание контейнеров внутри Pod
  - name: web             # Название контейнера
    image: travk/web:0.1  # Образ из которого создается контейнер
    ###readinessProbe###
    readinessProbe:       # Добавим проверку готовности
      httpGet:            # веб-сервера отдавать
        path: /index.html # контент
        port: 80
    ####################    
...
```
### Запустим под и убедимся, что он перешел в статус **Running**.
```bash
kubectl apply -f web-pod.yaml
kubectl get pod/web
```

![web-pod](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/web-pod.png)

### Посмотрим **Conditions** и **Events** в выводе **describe**.
```bash
kubectl describe pod/web
```

![web-pod-describe](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/web-pod-describe.png)

### Проверка не была пройдена, т.к наш веб сервер использует порт **8000**, а проверка настроена на **80**.
### Отложим исправление ошибки и добавим проверку **livenessProbe** на **8000** порт.
```yaml
...
spec:                     # Описание Pod
  containers:             # Описание контейнеров внутри Pod
  - name: web             # Название контейнера
    image: travk/web:0.1  # Образ из которого создается контейнер
    ###readinessProbe###
    readinessProbe:       # Добавим проверку готовности
      httpGet:            # веб-сервера отдавать
        path: /index.html # контент
        port: 80
    #################### 
    ###livenessProbe###
    livenessProbe:
      tcpSocket:
        port: 8000
    #################### 
...
```
### Запустим pod с новой проверкой.
```bash
kubectl apply -f web-pod.yaml 
```
### Вопрос для самопроверки:
1. Почему следующая конфигурация валидна, но не имеет
смысла?
```yaml
livenessProbe:
  exec:
    command:
      - 'sh'
      - '-c'
      - 'ps aux | grep my_web_server_process'
```
*Данная команда не имеет смысла, т.к проверяет наличие процесса, но не его работоспособность.*
1. Бывают ли ситуации, когда она все-таки имеет смысл?
*Если наличие процесса=работающее приложение*

# Создание Deployment
### Создадим папку **kubernetes-networks** и файл **web-deploy.yaml**.
```bash
mkdir kubernetes-networks
touch kubernetes-networks/web-deploy.yaml
```
### Заполним манифест **web-deploy.yaml**.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web      # Название нашего объекта Deployment
spec:
  replicas: 1    # Начнем с одного пода
  selector:      # Укажем, какие поды относятся к нашему Deployment:
    matchLabels: # - это поды с меткой
      app: web   # app и ее значением web
  template:      # Теперь зададим шаблон конфигурации пода
    metadata:
      labels:
        app: web
    spec:                     # Описание Pod
      containers:             # Описание контейнеров внутри Pod
      - name: web             # Название контейнера
        image: travk/web:0.1  # Образ из которого создается контейнер
        ###readinessProbe###
        readinessProbe:       # Добавим проверку готовности
          httpGet:            # веб-сервера отдавать
            path: /index.html # контент
            port: 80
        ####################
        ###livenessProbe###
        livenessProbe:
          tcpSocket:
            port: 8000
        ####################
        volumeMounts:
        - name: app
          mountPath: /app
      initContainers:
        - name: init-busybox
          image: busybox:1.32.0
          command: ['sh', '-c', 'wget -O- https://tinyurl.com/otus-k8s-intro | sh'] 
          #Command аналог ENTRYPOINT в Dockerfile.
          volumeMounts:
            - name: app
              mountPath: /app
      volumes:
        - name: app
          emptyDir: {}

```
### Удалим старый **pod** и развернём **deployment**.
```bash
kubectl delete pod/web --grace-period=0 --force
cd kubernetes-networks/
kubectl apply -f web-deploy.yaml
```
### Посмотрим **describe deployment**.
```bash
kubectl describe deployment web
```

![deployment-describe-fail](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/deployment-describe-fail.png)

В нашем манифесте, проверка идёт по 80 порту, из-за этого проверка завершается неудачно и деплоймент не может перейти в состояние **Available**.

### Исправим это, поменяем в файле web-deploy.yaml следующие параметры:
1. Увеличим число реплик до **3**.
2. Исправим порт **readinessProbe** на порт **8000**.

```yaml
***
spec:
  replicas: 3 
***
        readinessProbe:      
          httpGet:           
            path: /index.html
            port: 8000
***
```
### Применим манифест и посмотрим **Conditions** в **describe deployment**. 
```bash
kubectl apply -f web-deploy.yaml
kubectl describe deploy/web
```
![deployment-describe-ok](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/deployment-describe-ok.png)

Проверка был пройдена и условия (Conditions) **Available** и **Progressing** имеют значение **true**.

### Добавим в манифест **Strategy**.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web      # Название нашего объекта Deployment
spec:
  replicas: 3    # Начнем с одного пода
  selector:      # Укажем, какие поды относятся к нашему Deployment:
    matchLabels: # - это поды с меткой
      app: web   # app и ее значением web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 100%
      maxUnavailable: 0
***
```
### Попробоуем разные варианты деплоя с крайними значениями **maxSurge** и **maxUnavailable** (оба **0**, оба **100%**, **0** и **100%**)

За процессом можно понаблюдать с помощью **kubectl get events --watch** или установить и использовать **kubespy**.
(kubespy trace deploy)

```bash
curl -LO https://github.com/pulumi/kubespy/releases/download/v0.6.0/kubespy-v0.6.0-linux-amd64.tar.gz
tar xzvf kubespy-v0.6.0-linux-amd64.tar.gz kubespy
sudo install kubespy /usr/local/bin/kubespy
```
# Создание Service
* ClusterIP - выделяет для каждого сервиса IP-адрес из особого диапазона (этот адрес виртуален и даже не настраивается на cетевых интерфейсах)
* Когда под внутри кластера пытается подключиться к виртуальному IP-адресу сервиса, то нода, где запущен под меняет адрес получателя в сетевых пакетах на настоящий адрес пода.
* Нигде в сети, за пределами ноды, виртуальный ClusterIP не встречается.

### Создадим манифест **web-svc-cip.yaml** и применим его.
```bash
touch kubernetes-networks/web-svc-cip.yaml
kubectl apply -f web-svc-cip.yaml
```
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc-cip
spec:
  selector:
    app: web
  type: ClusterIP
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```
Проверим создался ли сервис.
```bash
kubectl get services
```

![web-svc-cip](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/web-svc-cip.png)

Подключимся к minikube ssh.
```bash
minikube ssh
sudo -i
curl http://10.98.30.236/index.html #Работает.
ip addr show                        #Нашего cluster ip нет.
iptables --list -nv -t nat          #Найдем наш cluster ip.
```

![cluster_ip_nat](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/cluster_ip_nat.png)

Более подробно про работу **cluster ip** можно [тут](https://msazure.club/kubernetes-services-and-iptables/) и [тут(ipvs) ](https://github.com/kubernetes/kubernetes/blob/master/pkg/proxy/ipvs/README.md)

# Включение IPVS

Попробуем включить IPVS в minikube (Доступен с версии 1.0.0).

```bash
kubectl --namespace kube-system edit configmap/kube-proxy
```

Изменим *mode:* **""** на *mode:* **"ipvs"**

Удалим под и дождемся его поднятия.

```bash
kubectl --namespace kube-system delete pod --selector='k8s-app=kube-proxy'
```
Проверим iptables в контейнере minikube.

```bash
minikube ssh
sudo -i
iptables --list -nv -t nat
```
cluster_ip_nat_del.png
![cluster_ip_nat_del](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/cluster_ip_nat_del.png)

Что-то поменялось, но старые цепочки на месте (хотя у них теперь 0 references)

**Kube-proxy** настроил все по-новому, но не удалил мусор.

## Полностью очистим все правила iptables:

Создадим minikube файл **/tmp/iptables.cleanup**

```bash

cat > /tmp/iptables.cleanup <<EOF
*nat
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
COMMIT
*filter
COMMIT
*mangle
COMMIT
EOF
```
Применим файл конфигурации.
```bash
iptables-restore /tmp/iptables.cleanup
iptables --list -nv -t nat
```
Дождёмся пока iptables восстановит правила.

![cluster_ip_nat_ipvs](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/cluster_ip_nat_ipvs.png)

Лишние правила были удалены. **Kube-proxy** периодически делает полную синхронизацию правил.

Проверим конфигурацию IPVS.
### Только для --vm-driver VM.
```bash
minikube ssh
toolbox #Запустим команду и окажемся в контейнере fedora.
dnf install -y ipvsadm && dnf clean all #установим ipvsadm
ipvsadm --list -n
```
![ipvs_list](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/ipvs_list.png)

Мы можем увидеть  **clusterip** нашего сервиса **10.105.251.88**

Пропингуем его

![ping_clusterip_ipvs](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/ping_clusterip_ipvs.png)

Пинг стал работать, всё дело в том, что этот ip привязан к интеррфейсу.Проверим это.

```bash
minikube ssh
ip addr show kube-ipvs0
```

![cluster_ip_addr](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/cluster_ip_addr.png)

# LoadBalancer и Ingress

## Устанока MetalLB
Если вы используете kube-proxy в режиме IPVS, начиная с Kubernetes v1.14.2, вам необходимо включить строгий режим ARP.

```bash
kubectl edit configmap -n kube-system kube-proxy
```
```yaml
...
    ipvs:
      strictARP: true
...
```
```bash
kubectl --namespace kube-system delete pod --selector=k8s-app=kube-proxy
```
Применим манифесты.
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
Проверим, что обьекты были созданы.

![metallb](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/metallb.png)

Выполним настройку.

Создадим файл **metallb-config.yaml** в папке **kubernetes-networks**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.17.255.1-172.17.255.255
```

В конфигурации мы настраиваем:
* Режим L2 (анонс адресов балансировщиков с помощью ARP).
* Создаем пул адресов 172.17.255.1-172.17.255.255 - они будут назначаться сервисам с типом LoadBalancer.

Применим наш манифест.
```bash
kubectl apply -f metallb-config.yaml
```
Сделаем копию файла web-svc-cip.yaml в web-svc-lb.yaml

```bash
cp web-svc-cip.yaml web-svc-lb.yaml
```

Изменим имя сервера и тип.

```yaml
...
metadata:
  name: web-svc-lb
spec:
  selector:
    app: web
  type: LoadBalancer
...
```
Применим манифест.
```bash
kubectl apply -f web-svc-lb.yaml
```
Посмотрим логи metallb контроллера и сервисы.
```bash
kubectl --namespace metallb-system logs controller-65db86ddc6-d4wl2
kubectl get svc
```

![ip_lb](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/ip_lb.png)

Для нашего сервиса быо назначен ip **172.17.255.1**.

Т.к наша сеть изолирована от основной ОС мы не может посмотреть в браузере наш сервис.

Добавим статический маршрут.

```bash
minikube ip # Узнаем ip minikube, это 10.10.10.194.
route add 172.17.255.0 mask 255.255.255.0 10.10.10.194 #Добавим маршрут.
```
Зайдя на адрес **http://172.17.255.1** мы увидим наше приложение, если обновить страницу **ctrl+f5** мы заметим, что периодически попадаем на разные поды.
По умолчанию в **IPVS** используется **Round-Robin** балансировка.

## DNS через MetalLB
* Сделайте сервис **LoadBalancer**, который откроет доступ к **CoreDNS** снаружи кластера (позволит получать записи через внешний IP). Например, **nslookup web.default.cluster.local** **172.17.255.10**.
* Поскольку **DNS** работает по **TCP** и **UDP** протоколам - учтите это в конфигурации. Оба протокола должны работать по одному и тому же IP-адресу балансировщика.
* Полученные манифесты положите в подкаталог ./coredns
  
 ```bash
 mkdir ./coredns
 ```
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: coredns-svc-lb-udp
  annotations:
    metallb.universe.tf/allow-shared-ip: coredns #Общая аннатация, для возможности использования одно IP в двух сервисах.
  namespace: kube-system
spec:
  selector:
    k8s-app: kube-dns #Лейбл пода core-dns.
  type: LoadBalancer
  loadBalancerIP: 172.17.255.10
  ports:
  - protocol: UDP
    port: 53
    targetPort: 53
---
apiVersion: v1
kind: Service
metadata:
  name: coredns-svc-lb-tcp
  annotations:
    metallb.universe.tf/allow-shared-ip: coredns
  namespace: kube-system
spec:
  selector:
    k8s-app: kube-dns
  type: LoadBalancer
  loadBalancerIP: 172.17.255.10
  ports:
  - protocol: TCP
    port: 53
    targetPort: 53
```
Проверим, что dns работает с локальном vm.
```bash
nslookup web-svc-cip.default.svc.cluster.local  172.17.255.10
```
![dns_metallb](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/dns_metallb.png)

## Создадим Ingress

Установим манифест ingress.
```bash
https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
```
В minikube можно установить ingress командой.
```bash
minikube addons enable ingress
```

Создадим сервис файл **nginx-lb.yaml** с конфигурацией **LoadBalancer**.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
```

Применим созданный манифест и посмотрим выданный IP.
```bash
kubectl apply -f nginx-lb.yaml
kubectl get svc -n ingress-nginx
```
Выполним curl по ip ingress nginx.
```bash
curl 172.17.255.2
```
![ingress_ip](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/ingress_ip.png)

Видим ответ дефолтного бекенда ingress nginx, значит всё работает.

* Наш Ingress-контроллер не требует ClusterIP для балансировки трафика.
* Список узлов для балансировки заполняется из ресурса Endpoints нужного сервиса (это нужно для "интеллектуальной" балансировки, привязки сессий и т.п.)
* Поэтому мы можем использовать headless-сервис для нашего веб-приложения.

Скопируем web-svc-cip.yaml в web-svc-headless.yaml
```bash
cp web-svc-cip.yaml web-svc-headless.yaml
```
* Изменим имя сервиса на **web-svc**
* Добавим параметр **clusterIP: None**

```yaml
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
  type: ClusterIP
  clusterIP: None
...
```
применим манифест и проверим, что **clusterIP** не назначен.

![headless_svc](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/headless_svc.png)

Создадим правило **Ingress** и назовем файл **web-ingress.yaml**.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: web
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /web
        backend:
          serviceName: web-svc
          servicePort: 8000
```
Применим манифест и убедимся, что всё создалось и работает.
```bash
kubectl apply -f web-ingress.yaml*
kubectl describe ingress/web
curl http://172.17.255.1//web/
```

![ingress_obj_ok](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/ingress_obj_ok.png)

По адресы **LB** мы успешно получили страничку, значит всё работает.

Балансировка при этом происходет на стороне **nginx**, а не средствами **IPVS**.

## ⭐ Добавим доступ к kubernetes-dashboard через наш Ingress-прокси и выполним следующие условия:

* Cервис должен быть доступен через префикс /dashboard).
* Kubernetes Dashboard должен быть развернут из официального манифеста. Актуальная ссылка есть в  [dashboard](https://github.com/kubernetes/dashboard).
* Написанный манифест положим в подкаталог ./dashboard

### Установим kubernetes-dashboard
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```
### Создадим ingress правила для дашборда.
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/configuration-snippet: |
      rewrite ^(/dashboard)$ $1/ redirect;
spec:
  rules:
    - http:
        paths:
          - path: /dashboard(/|$)(.*)
            backend:
              serviceName: kubernetes-dashboard
              servicePort: 443

```

Применим его и посмотрим доступность.

```bash
kubectl apply -f web-ingress-dashboard.yaml
```

![k8s_dashboard](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/k8s_dashboard.png)

Сервис доступен и работает.

## ⭐ Canary для Ingress
Реализуем canry деплой с помощью **ingress-nginx**.

* Перенаправление части трафика на выделенную группу подов должно происходить по HTTP-заголовку.
* Сделаем 2 пода с разными версиями.
* Манифесты положим в папку ./canary.
* Документация [тут](https://github.com/kubernetes/ingress-nginx/blob/master/docs/user-guide/nginx-configuration/annotations.md#canary).




```bash
nginx.ingress.kubernetes.io/canary: "true"      #Это означает, что Kubernetes не будет рассматривать этот Ingress как самостоятельный и пометит его как Canary, связав с основным Ingress.
nginx.ingress.kubernetes.io/canary-weight: "80" #Перенаправим 50% запросов на на canary сервис.
nginx.ingress.kubernetes.io/canary-by-header: "canary" #Установив header always или never мы можем определять, будет ли попадать трафик на canary сервис или нет.
```
### Передадим header **canary=always**
![canary_always](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/canary_always.png)

Все *100%* запросов уходят на **canary** сервис **deploy-2**.

### Передадим header **canary=never**
![canary_never](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-networks/canary_never.png)

Все *100%* запросов не попадают на **canary** сервис и уходят на **deploy-1**.