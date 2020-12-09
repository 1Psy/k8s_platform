# Задача 01
### 1. Создать Service Account bob, дать ему роль admin в рамках всего кластера
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bob
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bob-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin
subjects:
- kind: ServiceAccount
  name: bob
  namespace: default
```
### 2. Создать Service Account dave без доступа к кластеру
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dave
```
# Задача 02
### 1. Создать Namespace prometheus
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prometheus
```
### 2. Создать Service Account carol в этом Namespace
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: carol
  namespace: prometheus
```
### 3. Дать всем Service Account в Namespace prometheus возможность делать get, list, watch в отношении Pods всего кластера.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  namespace: prometheus
  name: pod-reader
rules:
- apiGroups: [""] 
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-sa-binding
  namespace: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader
subjects:
- kind: Group
  name: system:serviceaccounts:prometheus
  apiGroup: rbac.authorization.k8s.io
```

# Задача 03
### 1. Создать Namespace dev
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```
### 2. Создать Service Account jane в Namespace dev
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jane
  namespace: dev
```
### 3. Дать jane роль admin в рамках Namespace dev
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-admin
  namespace: dev
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: jane
    namespace: dev
```
### 4. Создать Service Account ken в Namespace dev
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ken
  namespace: dev
```
### 5. Дать ken роль view в рамках Namespace dev
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ken-view
  namespace: dev
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: ken
    namespace: dev
```