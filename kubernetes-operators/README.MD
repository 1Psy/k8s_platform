
## Minikube

Запустим minikube с использованием docker driver.

Производим запуск не из под root.
```bash
minikube start --driver=none
```
# CustomResource и CustomResourceDefifinition
## Что будем делать.
* Напишем CustomResource и CustomResourceDefinition для mysql оператора
* Напишем часть логики mysql оператора при помощи python KOPF
* Сделаем соберем образ и сделаем деплой оператора

Создадим директории **kubernetes-operators/deploy**
```bash
mkdir -p kubernetes-operators/deploy && cd kubernetes-operators
```

## CustomResource

Создадим  **CustomResource** в файле **deploy/cr.yaml** и применим его.
```yaml
apiVersion: otus.homework/v1
kind: MySQL
metadata:
  name: mysql-instance
spec:
  image: mysql:5.7
  database: otus-database
  password: otuspassword  # Так делать не нужно, следует использовать secret
  storage_size: 1Gi
usless_data: "useless info"
```
```bash
> kubectl apply -f deploy/cr.yaml
error: unable to recognize "deploy/cr.yaml": no matches for kind "MySQL" in version "otus.homework/v1"
```
Получим ошибку о том, что в k8s api нет обьектов с том MySQL.

Исправим это, создав **CustomResourceDefinition** - ресурс для определения **CustomResource**
## CustomREsourceDefinition

Создадим **CRD** **deploy/crd.yaml** и применим его.

Так же добавим в него схему для валидации.
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: mysqls.otus.homework # имя CRD должно иметь формат plural.group
spec:
  scope: Namespaced # Данный CRD будер работать в рамках namespace
  group: otus.homework # Группа, отражается в поле apiVersion CR
  versions: # Список версий
    - name: v1
      served: true # Будет ли обслуживаться API-сервером данная версия
      storage: true # Фиксирует  версию описания, которая будет сохраняться в etcd
      schema:
        openAPIV3Schema:
          type: object
          properties:
            apiVersion:
              type: string
            kind:
              type: string
            metadata:
              type: object
              properties:
                name:
                  type: string
            spec:
              type: object
              properties:
                image:
                  type: string
                database:
                  type: string
                password:
                  type: string
                storage_size:
                  type: string
  names: # различные форматы имени объекта CR
    kind: MySQL # kind CR
    plural: mysqls
    singular: mysql
    shortNames:
      - ms

```

Создадим **CRD** 
```bash
kubectl apply -f deploy/crd.yaml
customresourcedefinition.apiextensions.k8s.io/mysqls.otus.homework created
```

Создадим **CR**
```bash
kubectl apply -f deploy/cr.yaml
error: error validating "deploy/cr.yaml": error validating data: ValidationError(MySQL): unknown field "usless_data" in homework.otus.v1.MySQL; if you choose to ignore these errors, turn validation off with --validate=false
```
Мы получили ошибку т.к поле "usless_data" не описано в схеме и не прошло валидацию, удалим его из **cr.yaml**.
```yaml
apiVersion: otus.homework/v1
kind: MySQL
metadata:
  name: mysql-instance
spec:
  image: mysql:5.7
  database: otus-database
  password: otuspassword  # Так делать не нужно, следует использовать secret
  storage_size: 1Gi
```
Применим обновленный манифест.
```bash
> kubectl apply -f deploy/cr.yaml
mysql.otus.homework/mysql-instance created
```
Валдация пройдена и ошибок больше нет.

С создаными обьектами мы можем взаимодействовать через следующие команды.
```bash
kubectl get crd
kubectl get mysqls.otus.homework
kubectl describe mysqls.otus.homework mysql-instance
```

Чтобы мы не могли создать **CR** при отсуствии нужных нам полей в **spec**, укажем эти поля в **cr.yaml**

```yaml
...
            spec:
              type: object
              properties:
                image:
                  type: string
                database:
                  type: string
                password:
                  type: string
                storage_size:
                  type: string
              required:
                - image
                - database
                - password
                - storage_size
          required:
            - apiVersion
            - kind
            - metadata
```

Применим манифесты.
```bash
kubectl apply -f cdr.yaml
kubectl apply -f cr.yaml
```
Проверим работу cdr, удалив поле storage_size: 1Gi из cr.yaml
```bash
> kubectl apply -f deploy/cr.yaml
error: error validating "deploy/cr.yaml": error validating data: ValidationError(MySQL.spec): missing required field "storage_size" in homework.otus.v1.MySQL.spec; if you choose to ignore these errors, turn validation off with --validate=false
```

Валидация не прошла и мы получили ошибку, вернём поле назад.


# Operators

**Оператор** включает в себя **CustomResourceDefinition** и **сustom сontroller**.
* **CRD** содержит описание объектов **CR**
* **Контроллер** следит за объектами определенного типа, и осуществляет всю логику работы оператора

## Наш контроллер будет обрабатывать следующие события.
1) При создании объекта типа (kind: mySQL), он будет:
* Cоздавать PersistentVolume, PersistentVolumeClaim, Deployment, Service для mysql
* Создавать PersistentVolume, PersistentVolumeClaim для бэкапов базы данных, если их еще нет.
2) При удалении объекта типа (kind: mySQL), он будет:
* Пытаться восстановиться из бэкапа
* Удалять все успешно завершенные backup-job и restore-job
* Удалять PersistentVolume, PersistentVolumeClaim, Deployment, Service для mysql

Создадим в папке **kubernetes-operator/deploy** следующие манифесты.

service-account.yaml
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mysql-operator
```
role.yaml
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: null
  name: mysql-operator
rules:
- apiGroups:
  - '*'
  resources:
  - '*'
  verbs:
  - '*'
```
**role-binding.yaml**
```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: workshop-operator
subjects:
- kind: ServiceAccount
  name: mysql-operator
  namespace: default
roleRef:
  kind: ClusterRole
  name: mysql-operator
  apiGroup: rbac.authorization.k8s.io
```
**deploy-operator.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      name: mysql-operator
  template:
    metadata:
      labels:
        name: mysql-operator
    spec:
      serviceAccountName: mysql-operator
      containers:
        - name: operator
          image: "travk/mysql-operator:v0.1"
          imagePullPolicy: "Always"
```


Применим их.
```bash
kubectl apply -f service-account.yaml
kubectl apply -f role.yaml
kubectl apply -f role-binding.yaml
kubectl apply -f deploy-operator.yaml
```
Создадим **CR** и проверим, что применились **pvc**.
```bash
> kubectl apply -f cr.yaml
mysql.otus.homework/mysql-instance created
> kubectl get pvc
NAME                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
backup-mysql-instance-pvc   Bound    pvc-90b31153-d7ec-45c6-8954-59052245c9ad   1Gi        RWO            standard       8s
mysql-instance-pvc          Bound    pvc-5037d4d1-2fb4-40d1-b0be-d80b13c4f83a   1Gi        RWO            standard       8s
```
Заполним базу созданного mysql-instance.
```bash
export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
kubectl exec -it $MYSQLPOD -- mysql -u root -potuspassword -e "CREATE TABLE test ( id smallint unsigned not null auto_increment, name varchar(20) not null, constraint pk_example primary key (id) );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES ( null, 'some data' );" otus-database
kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "INSERT INTO test ( id, name ) VALUES ( null, 'some data-2' );" otus-database
```
Проверим содержимое.
```bash
> kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
```
Удалим mysql-instance.
```bash
> kubectl delete mysqls.otus.homework mysql-instance
mysql.otus.homework "mysql-instance" deleted
```
Теперь **kubectl get pv** показывает, что **PV** для **mysql** больше нет, а **kubectl get jobs.batch** показывает:
```bash
> kubectl get jobs.batch
NAME                         COMPLETIONS   DURATION   AGE
backup-mysql-instance-job    1/1           1s         3m1s
```
Создадим завово **CR mysql-instance**.
```bash
> kubectl apply -f cr.yaml
mysql.otus.homework/mysql-instance created
```
Подождем и посмотрим в базу.
```bash
export MYSQLPOD=$(kubectl get pods -l app=mysql-instance -o jsonpath="{.items[*].metadata.name}")
> kubectl exec -it $MYSQLPOD -- mysql -potuspassword -e "select * from test;" otus-database
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+-------------+
| id | name        |
+----+-------------+
|  1 | some data   |
|  2 | some data-2 |
+----+-------------+
```
Данные успешно восстановлены.
