# Установка и запуск kind.
## Установим kind.
https://kind.sigs.k8s.io/docs/user/quick-start/#installation
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.9.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /usr/local/bin/kind
```
![kind_install](https://github.com/1Psy/k8s_platform/blob/main/img/k8s_controllers/kind_install.png)

## Запустим kind.
```bash
kind create cluster
export KUBECONFIG="$(kind get kubeconfig-path --name="kind")"
```

# Minio
## Запуск StatefulSet

Применим манифест с minio.
```bash
kubectl apply -f kubernetes-volumes/minio-statefulset.yaml
```
После применения у нас создадутся следующие ресурсы:
 * **Pod** с **minio**
 * **PVC**
 * **PV** на основе **PVC** с помощь **default** **storageclass**

Проверим это с помощью команд.

```bash
kubectl get statefulsets
kubectl get pods
kubectl get pvc
kubectl get pv
kubectl describe <resource> <resource_name>
```
minio
![minio](https://github.com/1Psy/k8s_platform/blob/main/img/kubernetes-volumes/minio.png)

## Headless Service

Сделаем наш **StatefulSet** доступным внутри кластера.
```yaml
apiVersion: v1
kind: Service
metadata:
  name: minio
  labels:
    app: minio
spec:
  clusterIP: None
  ports:
    - port: 9000
      name: minio
  selector:
    app: minio
```

Применим манифест **Headless Service**.
```bash
kubectl apply -f kubernetes-volumes/minio-headless-service.yaml
```

## Добавим secrets ключи в отдельный файл.
Создадим файл **minio-secrets.yaml**.
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: minio
type: Opaque
data:
  MINIO_ACCESS_KEY: YWRtaW4=
  MINIO_SECRET_KEY: MWYyZDFlMmU2N2Rm
```
Пропишим его использование как переменные в **minio-statefulset.yaml**.
```yaml
...
          envFrom:
            - secretRef:
                name: minio
...
```
Применим созданные манифесты и перезапустим pod.

```bash
kubectl apply -f kubernetes-volumes/minio-secrets.yaml
kubectl apply -f kubernetes-volumes/minio-statefulset.yaml
```