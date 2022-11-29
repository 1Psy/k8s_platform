## Подготовка.
```bash
mkdir kubernetes-logging && cd kubernetes-logging
```
Создадим кластер k8s из 3х нод.

```bash
gcloud container clusters create cluster-vault \
--machine-type "n1-standard-2" \
--num-nodes "3" 
```
## Установка consul.
Склонируем репозиторий **consul** и установим его.
```bash
git clone https://github.com/hashicorp/consul-k8s.git
cd consul-k8s/charts/consul 
helm install consul .
```
## Установка vault.

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Создадим **values-notls.yaml** для **vault**.
```yaml
server:
  standalone:
    enabled: false
  ha:
    enabled: true
ui:
  enabled: true
```
Установим  **vault**.
```bash
helm install vault hashicorp/vault -f values-notls.yaml
```
### Инициализация vault.

Зайдем в под **vault-0** и выполним инициализацию.
```bash
# --key-shares - кол-во ключей, на которые нужно разделить мастер ключ.
# --key-threshold - кол-во ключей, необходимых для распечатки.

> kubectl exec -it vault-0 -- vault operator init --key-shares=1 --key-threshold=1
Unseal Key 1: UHHrYjVUjRm7noodomr9fnYToYe8f0iW9na+sgL6ff0=

Initial Root Token: s.Tuc7kKkNPRQHnMvtnm7dLssy
```
Проверим статус vault-0.
```bash
> kubectl exec -it vault-0 -- vault status
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       1
Threshold          1
Unseal Progress    0/1
Unseal Nonce       n/a
Version            1.6.2
Storage Type       consul
HA Enabled         true
```
Проверим переменную в контейнере **VAULT_ADDR**.
```bash
> kubectl exec -it vault-0 -- env | grep VAULT_ADDR
VAULT_ADDR=http://127.0.0.1:8200
```
### Распечатаем vault.
Зайдем в каждый под и введем команду с ключем, который мы полуили при инициализации.
```bash
kubectl exec -it vault-0 -- vault operator unseal 'UHHrYjVUjRm7noodomr9fnYToYe8f0iW9na+sgL6ff0='
kubectl exec -it vault-1 -- vault operator unseal 'UHHrYjVUjRm7noodomr9fnYToYe8f0iW9na+sgL6ff0='
kubectl exec -it vault-2 -- vault operator unseal 'UHHrYjVUjRm7noodomr9fnYToYe8f0iW9na+sgL6ff0='
```
Проверим статус vault.
```bash
> kubectl exec -it vault-0 -- vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.6.2
Storage Type    consul
Cluster Name    vault-cluster-b6f3496a
Cluster ID      be9cfcdc-861d-6aec-5093-009793aa302d
HA Enabled      true
HA Cluster      https://vault-0.vault-internal:8201
HA Mode         active
```

Статус **Sealed** изменился на **false**.

### Авторизация.
Проверим доступные авторизации.
```bash
> kubectl exec -it vault-0 -- vault auth list
Error listing enabled authentications: Error making API request.

URL: GET http://127.0.0.1:8200/v1/sys/auth
Code: 400. Errors:

* missing client token
```

Залогинимся в **vault** с помощью **root** ключа.
```bash
> kubectl exec -it vault-0 -- vault login
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.Tuc7kKkNPRQHnMvtnm7dLssy
token_accessor       T8EKCpozuIIY45Q0OPVDOq3i
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
```

Запросим список авторизаций еще раз.
```bash
> kubectl exec -it vault-0 -- vault auth list
Path      Type     Accessor               Description
----      ----     --------               -----------
token/    token    auth_token_6eeb3093    token based credentials
```

### Создадим секреты.
Включим хранилище секретов по пусти otus/
```bash
> kubectl exec -it vault-0 -- vault secrets enable --path=otus kv
Success! Enabled the kv secrets engine at: otus/
```
Посмотрим вывод, что у нас создалось и что уже существует.
```bash
> kubectl exec -it vault-0 -- vault secrets list --detailed
Path          Plugin       Accessor              Default TTL    Max TTL    Force No Cache    Replication    Seal Wrap    External Entropy Access    Options    Description                                                UUID
----          ------       --------              -----------    -------    --------------    -----------    ---------    -----------------------    -------    -----------                                                ----
cubbyhole/    cubbyhole    cubbyhole_52ccc453    n/a            n/a        false             local          false        false                      map[]      per-token private secret storage                           81f9c220-ea52-4c77-cf28-11d9b01842ba
identity/     identity     identity_4224b948     system         system     false             replicated     false        false                      map[]      identity store                                             dacbab45-15bd-8120-bfcd-fe73c6349726
otus/         kv           kv_9f5177bc           system         system     false             replicated     false        false                      map[]      n/a                                                        6c79c58a-06d1-ad84-39b7-89444554f7b8
sys/          system       system_0e26c009       n/a            n/a        false             replicated     false        false                      map[]      system endpoints used for control, policy and debugging    b0381ff9-480e-64c7-fedb-b8660985053d
```
Запишем два ключ в хранилище **otus/**.
```bash
> kubectl exec -it vault-0 -- vault kv put otus/otus-ro/config username='otus' password='Pass233333'
Success! Data written to: otus/otus-ro/config
> kubectl exec -it vault-0 -- vault kv put otus/otus-rw/config username='otus' password='Pass233333rw'
Success! Data written to: otus/otus-rw/config
```
Прочитаем созданные нами ключи двумя способами.
```bash
> kubectl exec -it vault-0 -- vault read otus/otus-ro/config
Key                 Value
---                 -----
refresh_interval    768h
password            Pass233333
username            otus
```
```bash
> kubectl exec -it vault-0 -- vault kv get otus/otus-rw/config
====== Data ======
Key         Value
---         -----
password    Pass233333rw
username    otus
```

Включим авторизацию через k8s.
```bash
> kubectl exec -it vault-0 -- vault auth enable kubernetes
Success! Enabled kubernetes auth method at: kubernetes/
```
Запросим список авторизаций.
```bash
> kubectl exec -it vault-0 -- vault auth list
Path           Type          Accessor                    Description
----           ----          --------                    -----------
kubernetes/    kubernetes    auth_kubernetes_5f66d87b    n/a
token/         token         auth_token_6eeb3093         token based credentials
```

Создадим *Service Account* **vault-auth**.
```bash
> kubectl create serviceaccount vault-auth
serviceaccount/vault-auth created
```

Создадим **ClusterRoleBinding** для созданного аккаунта и запишем это в файле **vault-auth-service-account.yaml**.

Который свяжет **sa** с ролью **auth-delegator** в рамках всего кластера.
```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1 
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io 
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: vault-auth
  namespace: default
```
Применим его.
```bash
> kubectl apply -f vault-auth-service-account.yaml
clusterrolebinding.rbac.authorization.k8s.io/role-tokenreview-binding created
```
Подготовим переменные для записи в конфиг кубер авторизации.
```bash
export VAULT_SA_NAME=$(kubectl get sa vault-auth -o jsonpath="{.secrets[*]['name']}")
export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)
export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)
export K8S_HOST=$(kubectl cluster-info | grep "Kubernetes control plane" | awk '/https/ {print $ NF}' | sed 's/\x1b\[[0-9;]*m//g')
#x1b Удалить все escape codes
```
Запишем конфиг авторизации в vault.
```bash
kubectl exec -it vault-0 -- vault write auth/kubernetes/config \
token_reviewer_jwt="$SA_JWT_TOKEN" \
kubernetes_host="$K8S_HOST" \
kubernetes_ca_cert="$SA_CA_CRT"

Success! Data written to: auth/kubernetes/config
```
Создадим файл политики, который определит доступ к секретам.
```bash
tee otus-policy.hcl <<EOF
path "otus/otus-ro/*" {
capabilities = ["read", "list"]
}
path "otus/otus-rw/*" {
capabilities = ["read", "create", "list"]
}
EOF
```
Создадим политику и свяжем с ролью.
```bash
kubectl cp otus-policy.hcl vault-0:./tmp
kubectl exec -it vault-0 -- vault policy write otus-policy /tmp/otus-policy.hcl
kubectl exec -it vault-0 -- vault write auth/kubernetes/role/otus bound_service_account_names=vault-auth bound_service_account_namespaces=default policies=otus-policy ttl=24h

Success! Data written to: auth/kubernetes/role/otus
```
Создадим под с привязанным сервис аккаунтом и установим туда curl и jq.
```bash
> kubectl run tmp --restart=Never --rm -i --tty --serviceaccount=vault-auth --image alpine:3.7
If you don't see a command prompt, try pressing enter.
/ # apk add curl jq
fetch http://dl-cdn.alpinelinux.org/alpine/v3.7/main/x86_64/APKINDEX.tar.gz
fetch http://dl-cdn.alpinelinux.org/alpine/v3.7/community/x86_64/APKINDEX.tar.gz
(1/6) Installing ca-certificates (20190108-r0)
(2/6) Installing libssh2 (1.9.0-r1)
(3/6) Installing libcurl (7.61.1-r3)
(4/6) Installing curl (7.61.1-r3)
(5/6) Installing oniguruma (6.6.1-r0)
(6/6) Installing jq (1.5-r5)
Executing busybox-1.27.2-r11.trigger
Executing ca-certificates-20190108-r0.trigger
OK: 7 MiB in 19 packages
```
Залогинимся в поде и получим клиентский токен.
```bash
VAULT_ADDR=http://vault:8200
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq
TOKEN=$(curl -k -s --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq '.auth.client_token' | awk -F\" '{print $2}')
```
```bash
VAULT_ADDR=http://vault:8200
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0ole": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq '.auth.client_token' | awk -F\" '{print $2}'){
  "request_id": "e9285e8c-1656-aa69-9d78-899fd95bdb91",
  "lease_id": "",
  "renewable": false,
  "lease_duration": 0,
  "data": null,
  "wrap_info": null,
  "warnings": null,
  "auth": {
    "client_token": "s.eK3wX6r8m5NPorRW1lPLjuSG",
    "accessor": "RJdGPzrbiqS55AUuKu9zfoJ5",
    "policies": [
      "default",
      "otus-policy"
    ],
    "token_policies": [
      "default",
      "otus-policy"
    ],
    "metadata": {
      "role": "otus",
      "service_account_name": "vault-auth",
      "service_account_namespace": "default",
      "service_account_secret_name": "vault-auth-token-nwqmt",
      "service_account_uid": "04c6d5de-20af-4c90-8549-bfab685314a8"
    },
    "lease_duration": 86400,
    "renewable": true,
100  1605  100   666  100   939  15857  22357 --:--:-- --:--:-- --:--:-- 38214
    "entity_id": "db0af90a-fae1-3b5a-05e3-8c13c5fdaca2",
    "token_type": "service",
    "orphan": true
  }
}
TOKEN=$(curl -k -s --request POST --data '{"jwt": "'$KUBE_TOKEN'", "role": "otus"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq '.auth.client_token' | awk -F\" '{print $2}')

echo $TOKEN
s.P9s23tThnIQdcGRXje0bdz28
```
Прочитаем секреты с помощью полученного нами токена.
```bash
curl --header "X-Vault-Token:s.P9s23tThnIQdcGRXje0bdz28" $VAULT_ADDR/v1/otus/otus-ro/config

t-Token:s.P9s23tThnIQdcGRXje0bdz28" $VAULT_ADDR/v1/otus/otus-rw/con{"request_id":"c275ddf7-8b59-a27d-6189-0ec53c6e51cb","lease_id":"","renewable":false,"lease_duration":2764800,"data":{"password":"Pass233333","username":"otus"},"wrap_info":null,"warnings":null,"auth":null}

curl --header "X-Vault-Token:s.P9s23tThnIQdcGRXje0bdz28" $VAULT_ADDR/v1/otus/otus-rw/config

{"request_id":"bb8df4bd-032d-d9e9-b7c9-8317c0bd150d","lease_id":"","renewable":false,"lease_duration":2764800,"data":{"password":"Pass233333rw","username":"otus"},"wrap_info":null,"warnings":null,"auth":null}
```
Попробуем записать данные.
```bash
curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:s.P9s23tThnIQdcGRXje0bdz28" $VAULT_ADDR/v1/otus/otus-ro/config
{"errors":["1 error occurred:\n\t* permission denied\n\n"]}

curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:s.P9s23tThnIQdcGRXje0bdz28" $VAULT_ADDR/v1/otus/otus-rw/config
{"errors":["1 error occurred:\n\t* permission denied\n\n"]}

curl --request POST --data '{"bar": "baz"}' --header "X-Vault-Token:s.P9s23tThnIQdcGRXje0bdz28" $VAULT_ADDR/v1/otus/otus-rw/config1/
```
* otus-ro/config - запись не удалась т.к capabilities = **["read", "list"]**
* otus-rw/config -  запись не удалась т.к capabilities = **["read", "create", "list"]**, мы можем читать создавать отображать данные, но не можем обнавлять уже созданные.
* otus-rw/config1 - всё успещно записалось т.к у нас есть права на **create**. 

Изменим политику в **otus-policy.hcl** на возможность обновлять существующие обьекты, добавив туда capabilities **"update"**.
```yaml
path "otus/otus-ro/*" {
capabilities = ["read", "list"]
}
path "otus/otus-rw/*" {
capabilities = ["read", "create", "update", "list"]
}
```
Обновим и применим политики.
```bash
kubectl cp otus-policy.hcl vault-0:./tmp
kubectl exec -it vault-0 -- vault policy write otus-policy /tmp/otus-policy.hcl

Success! Uploaded policy: otus-policy
```
Попробуем изменить **otus-rw/config**.

Выполним команду в поде.
```bash
curl --request POST --data '{"test": "test2"}' --header "X-Vault-Token:s.P9s23tThnIQdcGRXje0bdz28" $VAULT_ADDR/v1/otus/otus-rw/config
curl --request GET --header "X-Vault-Token:s.P9s23tThnIQdcGRXje0bdz28" $VAULT_ADDR/v1/otus/otus-rw/config

{"request_id":"f14bb161-92d0-fd9f-f1ad-8ebc1dde7a31","lease_id":"","renewable":false,"lease_duration":2764800,"data":{"test":"test2"},"wrap_info":null,"warnings":null,"auth":null}
```
Данные успешно были записаны.

### Получим секреты Vault-agent для страницы nginx.
* Авторизуемся через **vault-agent** и получим клиентский токен
* Достанем секрет из vault, сгенерируем агентом файл index.html используя **Consul Template**.
* Итог - nginx получил секрет из волта, не зная ничего про волт

Скачаем примеры.
```bash
git clone https://github.com/hashicorp/vault-guides.git
cd vault-guides/identity/vault-agent-k8s-demo
```
Изменим файл **example-k8s-spec.yaml**
```bash
mv example-k8s-spec.yaml k8s-spec.yaml
```
```yaml
...
      env:
        - name: VAULT_ADDR
          value: http://vault:8200 #Укажем адрес нашего vault.
...
```
Изменим файл **configmap.yaml**
```yaml
apiVersion: v1
data:
    vault-agent-config.hcl: |
        # Comment this out if running as sidecar instead of initContainer
        exit_after_auth = true

        pid_file = "/home/vault/pidfile"

        auto_auth {
            method "kubernetes" {
                mount_path = "auth/kubernetes"
                config = {
                    role = "otus"    #Наша роль для доступа к волту через sa.
                }
            }

            sink "file" {
                config = {
                    path = "/home/vault/.vault-token"
                }
            }
        }

        template {
        destination = "/etc/secrets/index.html"
        contents = <<EOT
        <html>
        <body>
        <p>Some secrets:</p>
        {{- with secret "otus/otus-ro/config" }}
        <ul>
        <li><pre>username: {{ .Data.username }}</pre></li> #Исправим формат данных.
        <li><pre>password: {{ .Data.password }}</pre></li> #Исправим формат данных.
        </ul>
        {{ end }}
        </body>
        </html>
        EOT
        }
kind: ConfigMap
metadata:
    name: example-vault-agent-config
    namespace: default
```
Применим.
```bash
kubectl apply -f configmap.yaml
kubectl apply -f k8s-spec.yaml
```
Зайдем pod и проверим, что отдаёт nginx.
```bash
kubectl exec -it vault-agent-example -- sh
```
```bash
curl localhost

<html>
<body>
<p>Some secrets:</p>
<ul>
<li><pre>username: otus</pre></li>       #Наш логин
<li><pre>password: Pass233333</pre></li> # и пароль из vault.
</ul>

</body>
</html>
```
### Создадим CA на базе vault.
Включим **pki secrets**.
```bash
kubectl exec -it vault-0 -- vault secrets enable pki
#Success! Enabled the pki secrets engine at: pki/
kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki
#Success! Tuned the secrets engine at: pki/
kubectl exec -it vault-0 -- vault write -field=certificate pki/root/generate/internal common_name="exmaple.ru" ttl=87600h > CA_cert.crt
```
Пропишем **urls** для **CA** и отозванных сертификатов.
```bash
kubectl exec -it vault-0 -- vault write pki/config/urls \
issuing_certificates="http://vault:8200/v1/pki/ca" \
crl_distribution_points="http://vault:8200/v1/pki/crl"
#Success! Data written to: pki/config/urls
```
Cоздадим промежуточный сертификат.
```bash
kubectl exec -it vault-0 -- vault secrets enable --path=pki_int pki
#Success! Enabled the pki secrets engine at: pki_int/
kubectl exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki_int
#Success! Tuned the secrets engine at: pki_int/
kubectl exec -it vault-0 -- vault write -format=json pki_int/intermediate/generate/internal \
common_name="example.ru Intermediate Authority" | jq -r '.data.csr' > pki_intermediate.csr
```
Пропишем промежуточный сертификат в **vault**.
```bash
kubectl cp pki_intermediate.csr vault-0:./tmp
kubectl exec -it vault-0 -- vault write -format=json pki/root/sign-intermediate \
csr=@/tmp/pki_intermediate.csr \
format=pem_bundle ttl="43800h" | jq -r '.data.certificate' > intermediate.cert.pem

kubectl cp intermediate.cert.pem vault-0:./tmp
kubectl exec -it vault-0 -- vault write pki_int/intermediate/set-signed \
certificate=@/tmp/intermediate.cert.pem
#Success! Data written to: pki_int/intermediate/set-signed
```

Создадим роль для выдачи сертификатов.
```bash
kubectl exec -it vault-0 -- vault write pki_int/roles/example-dot-ru \
allowed_domains="example.ru" allow_subdomains=true max_ttl="720h"
#Success! Data written to: pki_int/roles/example-dot-ru
```
Создадим и отзовем сертификат.
```bash
> kubectl exec -it vault-0 -- vault write pki_int/issue/example-dot-ru common_name="shitlab.example.ru" ttl="24h"
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUYtTGuV0LlRDOZHA6V5ZOlysk8hIwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMTAyMDcxNDAyNTJaFw0yNjAy
MDYxNDAzMjJaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANYog6J1SH8+
WWUJJohbOslUUhwoRRRHYRlXWZJB5PTia2BJzJrHnsiD71qiTB3quvlaxxrfz/qj
fjJIC3lvEIxyZvGAPSrY6Frp1gUEo7sUZi/sFMq62F29W9mFB8Cdkz/dGF+1jLAd
BpKcxHxt7PWDhKFQoFCxc9xfH6+LYvYr/WGJpD222YEblrl9AdGfmxbKQiQjehnG
tIlBwcEgc8QS5HetxI/6RzOaeLBgYvz/uMWJ5QeIHyPvEsyYzqbjrRTxemuxHxW+
HfKSrl2/shWVAwpbZHXamXNtynJiMJs7Ud7nRoJO/RLt9gtoOKAJz/yZx/BsV9zo
e6uJEpQEU5ECAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUiE55qDVpcZ65mqV+/mqglfjtSxowHwYDVR0jBBgwFoAU
WDqhr9KmSr8S54HfmgGNtxNBTpowNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
HcqtjZUg8ldqaYmIKF6Ulo2bRJxTctW6gYsFhd5St1RxM6X/OH7c+ub3P+b0OheC
wmv9BmjVqvGExYBd/kd6Zl/HPeXFhN45kJ0ptVrPisIt0MKFJOxoWEvdBbc8j6uC
jaSGCjgykN1Xry/mnSvB+X6TPlq97kUpbmEHQDSaRabkmA1eKMDJh8bZPv/AVrHq
Qjo9oJaV6VB9VWbs8zsYDHuTMlWnwe8NH+VKxDcNUpBsXNNFXIfJOhnGdng8CR0B
W8WJYzOmxr1y9plUFnhGsOBubM3vZdjnd0UfLI66j7yJP+WZ7ETZQegf+yk8QsTV
hVrzMFPGqLp8tv/X/xbmuA==
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDaTCCAlGgAwIBAgIUdLkI0X+DTJytpv8iVMbljATEV7owDQYJKoZIhvcNAQEL
BQAwLDEqMCgGA1UEAxMhZXhhbXBsZS5ydSBJbnRlcm1lZGlhdGUgQXV0aG9yaXR5
MB4XDTIxMDIwNzE0MDYzNVoXDTIxMDIwODE0MDcwNVowHTEbMBkGA1UEAxMSc2hp
dGxhYi5leGFtcGxlLnJ1MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA
n/VWIxcRqFOpFAoRc5TzZGe/2Xb9tJ9PEilQJUhEGI72tIlH+OOROhH//IdBgAVY
AInwOpOalVhPK0rO2D04MOJHuLQnYSIUEK8BqQZuIx+Vn/rIGMvuDug/X+VV/fDH
gmEDgNtpNpws4StTiQJTkbRZ/cQ471CPAen6UgxNcABZJSZWjv1+OtRQqBcgw/w4
kmxzgkItVkR14ykO8OJ3KAwSJ8XdrmRuICsxcc1PXH7NYBFxwVjLuZKPtakzyJag
XZLfYNq0doXrA53U2A4A2utDfKmC/vCLgAbPV8aCf9CLIyX+7Gb0NiFLHeb8UL8B
nLXFXp9URfzUMuM7/SESIwIDAQABo4GRMIGOMA4GA1UdDwEB/wQEAwIDqDAdBgNV
HSUEFjAUBggrBgEFBQcDAQYIKwYBBQUHAwIwHQYDVR0OBBYEFFz4GAbKSjQA9rz2
asjbu82pK4mxMB8GA1UdIwQYMBaAFIhOeag1aXGeuZqlfv5qoJX47UsaMB0GA1Ud
EQQWMBSCEnNoaXRsYWIuZXhhbXBsZS5ydTANBgkqhkiG9w0BAQsFAAOCAQEAbs4O
YEDVwWoS2M9pAFkN2paeJfB4eWCIa0gELcgaPMxDGFgq4TojaDQP/lBd3R6gt3jl
/mEKz9ZPiEGYGqni1ZYRrhtnyUYM2OCd5S6igzwU3WaWHrlh3Jr+YhiVP7mTwZAZ
D4WmbayYysACuw02NPCJgCPmT09WoKI9kAFf5LAz+MZI4Dm0BPdrnAA3GNKUJBk+
wIIOHgFL4jFrQMzMCI//BPgPZm336ULoJbf/59jknPVcxu2ssaKeLlEMyGZgjMDn
vShq0lKi0mwQE/cZz9jiNG04FjwrrMRiHYsxZd9m3XwBQEPpfU90dqGvOXTF8mZ9
cqKgHnUUmDRy/4GL3Q==
-----END CERTIFICATE-----
expiration          1612793225
issuing_ca          -----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUYtTGuV0LlRDOZHA6V5ZOlysk8hIwDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMTAyMDcxNDAyNTJaFw0yNjAy
MDYxNDAzMjJaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBANYog6J1SH8+
WWUJJohbOslUUhwoRRRHYRlXWZJB5PTia2BJzJrHnsiD71qiTB3quvlaxxrfz/qj
fjJIC3lvEIxyZvGAPSrY6Frp1gUEo7sUZi/sFMq62F29W9mFB8Cdkz/dGF+1jLAd
BpKcxHxt7PWDhKFQoFCxc9xfH6+LYvYr/WGJpD222YEblrl9AdGfmxbKQiQjehnG
tIlBwcEgc8QS5HetxI/6RzOaeLBgYvz/uMWJ5QeIHyPvEsyYzqbjrRTxemuxHxW+
HfKSrl2/shWVAwpbZHXamXNtynJiMJs7Ud7nRoJO/RLt9gtoOKAJz/yZx/BsV9zo
e6uJEpQEU5ECAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUiE55qDVpcZ65mqV+/mqglfjtSxowHwYDVR0jBBgwFoAU
WDqhr9KmSr8S54HfmgGNtxNBTpowNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
HcqtjZUg8ldqaYmIKF6Ulo2bRJxTctW6gYsFhd5St1RxM6X/OH7c+ub3P+b0OheC
wmv9BmjVqvGExYBd/kd6Zl/HPeXFhN45kJ0ptVrPisIt0MKFJOxoWEvdBbc8j6uC
jaSGCjgykN1Xry/mnSvB+X6TPlq97kUpbmEHQDSaRabkmA1eKMDJh8bZPv/AVrHq
Qjo9oJaV6VB9VWbs8zsYDHuTMlWnwe8NH+VKxDcNUpBsXNNFXIfJOhnGdng8CR0B
W8WJYzOmxr1y9plUFnhGsOBubM3vZdjnd0UfLI66j7yJP+WZ7ETZQegf+yk8QsTV
hVrzMFPGqLp8tv/X/xbmuA==
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAn/VWIxcRqFOpFAoRc5TzZGe/2Xb9tJ9PEilQJUhEGI72tIlH
+OOROhH//IdBgAVYAInwOpOalVhPK0rO2D04MOJHuLQnYSIUEK8BqQZuIx+Vn/rI
GMvuDug/X+VV/fDHgmEDgNtpNpws4StTiQJTkbRZ/cQ471CPAen6UgxNcABZJSZW
jv1+OtRQqBcgw/w4kmxzgkItVkR14ykO8OJ3KAwSJ8XdrmRuICsxcc1PXH7NYBFx
wVjLuZKPtakzyJagXZLfYNq0doXrA53U2A4A2utDfKmC/vCLgAbPV8aCf9CLIyX+
7Gb0NiFLHeb8UL8BnLXFXp9URfzUMuM7/SESIwIDAQABAoIBAByh2rhPQc7ioso+
fx80U1jupU202fcB6TJ0CK+TvvSyaoCobiSj/3PokuzN/TCARF9cUY76qtn+w7UZ
H+tfEfbWGkZ6oO3nHrZZzS9HQmGxHqRTTIv7vPIYKnfs4jahaeTxnqHrJGstC2+G
xN4qd3EGWnFtNU/JGMAb9L3JC/d3IApCPbfEkExbde8oUAvSPrCytL57MwiYjVd1
BJS++8AxgfXz8u7vbio6bO53BzdLhsnE5GABfeX6z4f784Dbvw1Bij+KVSAgcLCv
P0DEAjCZECJbnMfOzNVnzQa/Di4xT0tJas8paNq+xqUSEC95T5XIQj+h25PdeSuG
1x9CG4kCgYEAzR1wiotq8bMYnDm+mFdGp6rtoCMODs88X/9eYXcBlbT0jla27XI9
WwzUfzmG2jI9Ezq+l1oCuZ9DYU322j4vr9ZQErxYxlMjBo1ocXeuS2+kV4ml41GC
lOqBpDqwo/4tFta5qJn58mTQEkQsF22h/+aAcs4b+WnblQamOQhYx6cCgYEAx6QP
99VYoWqEo+Xcxd+6uhATn7Erk03jL1tDHYSf53f/DGpipu+GnH/ZY0SxcEqloPpc
sSO2DFpojLn6l+907gTLASrr7fQg0V1ES1l+ermdR3xOH3BsRNXLpU7Lm9VLm6dB
PFGXfsUgZYZJf6mfOGs1K0lNzEF4+PhBwTPo8SUCgYEAwmR3R2aQSVciKNV1M+3r
EXamVFFEvnZbUuOz8A0YXGiuKcQ3A9x0F7mWHqlHi713MqVHAiwVTxSIiEyaegXb
rfsf0lHclMEv/cLAzGDPt8Cr8SaMKF8Usct6aEoTJKj81JtWQ1sCjl5K8DTZ0K5f
jSZIcAW0b3GK4Lb3jC/D7BsCgYATpe/0Ykjwyfz37H2H8/b6rO5hfxxcyQOc+4xl
cVITg9nULPw8lcJX0LVicbZM6OvYJgvNW0hlethjUUJ6hZSqgEpCeSjzKF26fUVl
bQ9noLD8Ah3mNjhRpgD+6fxb/+gLD7D58MXKMjh4BXliWR1Ufiijc/l2KtNvHW/L
RgKdCQKBgCzmOco9Z0U851vXq5Bu0r5F2TiFlW9faFA4a/kUGRsFLQeZqwKAJSLt
sOLiYfX87lQX/cfmkt//dQFjLcvLe4nS4dvwMMAsDZx395yIPYmVyWVj18RA3IDq
XMCyJiaw0JJ4OjNDbTIFlc8o6+WTr/NNEN8sDlpTALRDIY17ar8R
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       74:b9:08:d1:7f:83:4c:9c:ad:a6:ff:22:54:c6:e5:8c:04:c4:57:ba
```
```bash
> kubectl exec -it vault-0 -- vault write pki_int/revoke serial_number="74:b9:08:d1:7f:83:4c:9c:ad:a6:ff:22:54:c6:e5:8c:04:c4:57:ba"
Key                        Value
---                        -----
revocation_time            1612706892
revocation_time_rfc3339    2021-02-07T14:08:12.514763239Z
```
# Включим vault в режиме TLS.
* https://www.vaultproject.io/docs/platform/k8s/helm/examples/standalone-tls
* https://cogarius.medium.com/a-vault-for-all-your-secrets-full-tls-on-kubernetes-with-kv-v2-c0ecd42853e1

 
 #### Создадим ключи сертификаты через CA K8S.

 Обьявим переменные.
 ```bash
 # SERVICE is the name of the Vault service in Kubernetes.
# It does not have to match the actual running service, though it may help for consistency.
SERVICE=vault

# NAMESPACE where the Vault service is running.
NAMESPACE=default

# SECRET_NAME to create in the Kubernetes secrets store.
SECRET_NAME=vault-server-tls

# TMPDIR is a temporary working directory.
TMPDIR=/tmp
```
Cоздадим ключ для подписи K8S.
```bash
openssl genrsa -out ${TMPDIR}/vault.key 2048
```
Создадим запрос на подпись сертификата.(CSR)
```bash
cat <<EOF >${TMPDIR}/csr.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${SERVICE}
DNS.2 = ${SERVICE}.${NAMESPACE}
DNS.3 = ${SERVICE}.${NAMESPACE}.svc
DNS.4 = ${SERVICE}.${NAMESPACE}.svc.cluster.local
IP.1 = 127.0.0.1
EOF
```
```bash
openssl req -new -key ${TMPDIR}/vault.key -subj "/CN=${SERVICE}.${NAMESPACE}.svc" -out ${TMPDIR}/server.csr -config ${TMPDIR}/csr.conf
```
Создадим манифест запроса.
```bash
export CSR_NAME=vault-csr #Название обьекта CSR для K8S
cat <<EOF >${TMPDIR}/csr.yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: ${CSR_NAME}
spec:
  groups:
  - system:authenticated
  request: $(cat ${TMPDIR}/server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```
Применим его и проверим, что запрос создалсяю
```bash
kubectl create -f ${TMPDIR}/csr.yaml
certificatesigningrequest.certificates.k8s.io/vault-csr created

kubectl get csr ${CSR_NAME}
NAME        AGE   REQUESTOR                    CONDITION
vault-csr   16s   ok@gmail.com                  Pending
```
Подтвердим запрос CSR.
```bash
kubectl certificate approve ${CSR_NAME}
certificatesigningrequest.certificates.k8s.io/vault-csr approved
```

Получим подписанный сертификат.
```bash
serverCert=$(kubectl get csr ${CSR_NAME} -o jsonpath='{.status.certificate}')
```
Экспортируем его в файл.
```bash
echo "${serverCert}" | openssl base64 -d -A -out ${TMPDIR}/vault.crt
```
Экспортируем CA K8S.
```bash
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d > ${TMPDIR}/vault.ca
```
Сохраним ключи и сертификаты с secret k8s.
```bash
kubectl create secret generic ${SECRET_NAME} \
        --namespace ${NAMESPACE} \
        --from-file=vault.key=${TMPDIR}/vault.key \
        --from-file=vault.crt=${TMPDIR}/vault.crt \
        --from-file=vault.ca=${TMPDIR}/vault.ca
```
#### For injector
```bash
# SERVICE is the name of the Vault agent injector service
SERVICE=vault-agent-injector-svc
# NAMESPACE where the Vault service is running.
NAMESPACE=default
# SECRET_NAME to create in the Kubernetes secrets store.
SECRET_NAME=vault-agent-injector-tls
# TMPDIR is a temporary working directory.
TMPDIR=/tmp
# Generate key for the certificate
$ openssl genrsa -out ${TMPDIR}/vault-injector.key 2048
# Create a file ${TMPDIR}/csr-vault-agent-injector.conf 
# with the following contents:
$ cat <<EOF >${TMPDIR}/csr-vault-agent-injector.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${SERVICE}
DNS.2 = ${SERVICE}.${NAMESPACE}
DNS.3 = ${SERVICE}.${NAMESPACE}.svc
DNS.4 = ${SERVICE}.${NAMESPACE}.svc.cluster.local
IP.1 = 127.0.0.1
EOF
# Create a Certificate Signing Request (CSR).
$ openssl req -new -key ${TMPDIR}/vault-injector.key -subj "/CN=${SERVICE}.${NAMESPACE}.svc" -out ${TMPDIR}/server-vault-agent-injector.csr -config ${TMPDIR}/csr-vault-agent-injector.conf
$ export CSR_NAME=vault-agent-injector-csr
$ cat <<EOF >${TMPDIR}/agent-injector-csr.yaml
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: ${CSR_NAME}
spec:
  groups:
  - system:authenticated
  request: $(cat ${TMPDIR}/server-vault-agent-injector.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
# Send the CSR to Kubernetes.
$ kubectl create -f ${TMPDIR}/agent-injector-csr.yaml
# Approve the CSR in Kubernetes
$ kubectl certificate approve ${CSR_NAME}
# Store key, cert, and Kubernetes CA into Kubernetes secrets store
# Retrieve the certificate.
$ serverCert=$(kubectl get csr ${CSR_NAME} -o jsonpath='{.status.certificate}')
# Write the certificate out to a file.
$ echo "${serverCert}" | openssl base64 -d -A -out ${TMPDIR}/vault-injector.crt
# Retrieve Kubernetes CA.
$ kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 --decode > ${TMPDIR}/vault-injector.ca
# for the helm chart we need to base64 encode the ca certificate 
# to fill the value injector.certs.caBundle you can run
# cat ${TMPDIR}/vault-injector.ca | base64
# Store the key, cert, and Kubernetes CA into Kubernetes secrets.
$ kubectl create secret generic ${SECRET_NAME} \
        --namespace ${NAMESPACE} \
        --from-file=vault-injector.key=${TMPDIR}/vault-injector.key \
        --from-file=vault-injector.crt=${TMPDIR}/vault-injector.crt \
        --from-file=vault-injector.ca=${TMPDIR}/vault-injector.ca
```

 #### Настроим values Helm chart Vault.

Создадим **values-tls.yaml** для **vault**.
```yaml
global:
  enabled: true
  tlsDisable: false

server:
  extraEnvironmentVars:
    VAULT_CACERT: /vault/userconfig/vault-server-tls/vault.ca

  extraVolumes:
    - type: secret
      name: vault-server-tls # Matches the ${SECRET_NAME} from above

  standalone:
    enabled: false
  ha:
    enabled: true

    config: |
      ui = true

      listener "tcp" {
        tls_disable = 0
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        tls_cert_file = "/vault/userconfig/vault-server-tls/vault.crt"
        tls_key_file  = "/vault/userconfig/vault-server-tls/vault.key"
        tls_client_ca_file = "/vault/userconfig/vault-server-tls/vault.ca"  
      }
      storage "consul" {
        path = "vault"
        address = "HOST_IP:8500"
      }

      service_registration "kubernetes" {}

      # Example configuration for using auto-unseal, using Google Cloud KMS. The
      # GKMS keys must already exist, and the cluster must have a service account
      # that is authorized to access GCP KMS.
      #seal "gcpckms" {
      #   project     = "vault-helm-dev-246514"
      #   region      = "global"
      #   key_ring    = "vault-helm-unseal-kr"
      #   crypto_key  = "vault-helm-unseal-key"
      #}

ui:
  enabled: true
  serviceType: "LoadBalancer"
```

#### Установим  **consul** и **vault**.
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
```
```bash
helm install consul hashicorp/consul \
--set global.name=consul \
--version 0.29.0
```
```bash
helm upgrade --install vault hashicorp/vault \
--version 0.9.1 \
-f values-tls.yaml
```

#### Инициализация.
Зайдем в под и выполним инициализацию.
```bash
> kubectl exec -it vault-0 -- vault operator init --key-shares=1 --key-threshold=1
Unseal Key 1: YiMh+fHlKhwnmlc+jWHA8L+QqjpTJWOtujnMJ0rRwJg=

Initial Root Token: s.GjUfqwomqdAkgu1gzGF3cgyI
```
Откроем vaults.
```bash
kubectl exec -it vault-0 -- vault operator unseal 'YiMh+fHlKhwnmlc+jWHA8L+QqjpTJWOtujnMJ0rRwJg='
kubectl exec -it vault-1 -- vault operator unseal 'YiMh+fHlKhwnmlc+jWHA8L+QqjpTJWOtujnMJ0rRwJg='
kubectl exec -it vault-2 -- vault operator unseal 'YiMh+fHlKhwnmlc+jWHA8L+QqjpTJWOtujnMJ0rRwJg='
```
