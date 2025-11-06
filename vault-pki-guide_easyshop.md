# Развертывание HashiCorp Vault как центра сертификации в Kubernetes

> **DevOps проект**: Интеграция PKI и управления секретами с автоматической выдачей сертификатов

## Содержание

1. [Введение и архитектура](#введение)
2. [Установка HashiCorp Vault](#установка-vault)
3. [Инициализация и unsealing](#инициализация-vault)
4. [Настройка PKI Engine](#настройка-pki)
5. [Интеграция с cert-manager](#интеграция-cert-manager)
6. [Автоматическая выдача сертификатов](#автоматическая-выдача)
7. [Политики доступа](#политики-доступа)
8. [Мониторинг и логирование](#мониторинг)
9. [Backup и восстановление](#backup)
10. [Advanced конфигурации](#advanced-конфигурации)
11. [Production deployment](#production-deployment)
12. [Disaster Recovery](#disaster-recovery)
13. [Интеграция с внешними системами](#интеграция-внешних-систем)

---

## Введение

### Цель проекта

Развертывание HashiCorp Vault в качестве корпоративного центра сертификации (PKI) для автоматической выдачи и управления SSL/TLS сертификатами в Kubernetes кластере.

### Преимущества Vault PKI

- ✅ **Автоматизация**: Выдача и обновление сертификатов без ручного вмешательства
- ✅ **Безопасность**: Централизованное управление секретами и приватными ключами
- ✅ **Масштабируемость**: Поддержка тысяч сертификатов
- ✅ **Аудит**: Полное логирование всех операций
- ✅ **Интеграция**: Встроенная поддержка cert-manager и Kubernetes
- ✅ **Compliance**: Соответствие требованиям безопасности и стандартам
- ✅ **Multi-tenancy**: Изоляция доступа между командами и проектами

### Архитектура решения

```
┌─────────────────────────────────────────────────┐
│              Kubernetes Cluster                 │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │         HashiCorp Vault                  │  │
│  │  ┌────────────┐  ┌──────────────────┐   │  │
│  │  │ Root CA    │→ │ Intermediate CA  │   │  │
│  │  │ (Offline)  │  │   (Online PKI)   │   │  │
│  │  └────────────┘  └──────────────────┘   │  │
│  │         ↓                  ↓              │  │
│  │  ┌────────────────────────────────────┐  │  │
│  │  │     Vault PKI Secret Engine        │  │  │
│  │  └────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────┘  │
│                      ↕                          │
│  ┌──────────────────────────────────────────┐  │
│  │          cert-manager                    │  │
│  │  ┌────────────────────────────────────┐  │  │
│  │  │  Vault Issuer Configuration        │  │  │
│  │  └────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────┘  │
│                      ↕                          │
│  ┌──────────────────────────────────────────┐  │
│  │         Applications & Services          │  │
│  │                                          │  │
│  │  • Ingress Controllers (TLS)            │  │
│  │  • Microservices (mTLS)                 │  │
│  │  • Databases (Encrypted connections)    │  │
│  │  • ArgoCD, Grafana, Jenkins (HTTPS)    │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Компоненты инфраструктуры

| Компонент | Версия | Назначение |
|-----------|--------|------------|
| HashiCorp Vault | 1.15+ | PKI Engine и управление секретами |
| cert-manager | 1.13+ | Автоматическая выдача сертификатов |
| Longhorn | 1.5+ | Persistent storage для Vault |
| Traefik | 2.10+ | Ingress с автоматическим TLS |

---

## Установка HashiCorp Vault

### Подготовка окружения

С jumphost выполните:

```bash
# Проверка доступности кластера
kubectl cluster-info
kubectl get nodes

# Создание namespace
kubectl create namespace vault

# Добавление Helm репозитория
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```

### Создание конфигурации Vault

Создайте файл `vault-values.yaml`:

```bash
cat > /tmp/vault-values.yaml <<'EOF'
# Режим standalone для dev/test или HA для production
server:
  # Образ Vault
  image:
    repository: hashicorp/vault
    tag: "1.15.4"
  
  # Ресурсы
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 512Mi
      cpu: 500m
  
  # Хранилище данных
  dataStorage:
    enabled: true
    size: 10Gi
    storageClass: longhorn
    accessMode: ReadWriteOnce
  
  # Хранилище для аудит логов
  auditStorage:
    enabled: true
    size: 5Gi
    storageClass: longhorn
    accessMode: ReadWriteOnce
  
  # Standalone режим (для dev/test)
  standalone:
    enabled: true
    config: |
      ui = true
      
      listener "tcp" {
        tls_disable = 1
        address = "[::]:8200"
        cluster_address = "[::]:8201"
      }
      
      storage "file" {
        path = "/vault/data"
      }
      
      # Telemetry для мониторинга
      telemetry {
        prometheus_retention_time = "30s"
        disable_hostname = true
      }

  # HA режим (для production)
  ha:
    enabled: false
    replicas: 3
    raft:
      enabled: true
      setNodeId: true
      config: |
        ui = true
        
        listener "tcp" {
          tls_disable = 1
          address = "[::]:8200"
          cluster_address = "[::]:8201"
        }
        
        storage "raft" {
          path = "/vault/data"
        }
        
        service_registration "kubernetes" {}

  # Service configuration
  service:
    enabled: true
    type: ClusterIP
    port: 8200
    targetPort: 8200

  # Ingress для Web UI
  ingress:
    enabled: false  # Создадим отдельно

# UI Configuration
ui:
  enabled: true
  serviceType: ClusterIP

# Injector для автоматического внедрения секретов в pods
injector:
  enabled: true
  
  resources:
    requests:
      memory: 256Mi
      cpu: 250m
    limits:
      memory: 512Mi
      cpu: 500m
EOF
```

### Установка Vault

```bash
# Установка через Helm
helm install vault hashicorp/vault \
  --namespace vault \
  --values /tmp/vault-values.yaml

# Ожидание готовности
kubectl -n vault wait --for=condition=ready pod -l app.kubernetes.io/name=vault --timeout=300s

# Проверка статуса
kubectl -n vault get pods
kubectl -n vault get pvc
```

### Создание Ingress для Vault UI

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vault-ingress
  namespace: vault
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
spec:
  rules:
  - host: vault.local.lab
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vault
            port:
              number: 8200
EOF

# Проверка Ingress
kubectl -n vault get ingress
```

### Добавление DNS записи

```bash
# На dns-server
ssh admin@dns-server.local.lab

sudo bash -c 'cat >> /etc/bind/zones/db.local.lab <<EOL
vault           IN      A       192.168.100.100
EOL'

# Увеличить Serial в db.local.lab
sudo nano /etc/bind/zones/db.local.lab
# Изменить Serial с N на N+1

# Перезагрузить зону
sudo rndc reload local.lab

exit

# Проверка DNS
dig @192.168.100.53 vault.local.lab +short
# Ожидаем: 192.168.100.100
```

---

## Инициализация Vault

### Инициализация и получение ключей

```bash
# Exec в Vault pod
kubectl -n vault exec -it vault-0 -- sh

# Инициализация Vault (выполняется один раз!)
vault operator init -key-shares=5 -key-threshold=3

# ВАЖНО: Сохраните output в безопасном месте!
# Вы получите:
# - 5 unseal keys
# - 1 root token
```

**Пример output:**

```
Unseal Key 1: base64_encoded_key_1
Unseal Key 2: base64_encoded_key_2
Unseal Key 3: base64_encoded_key_3
Unseal Key 4: base64_encoded_key_4
Unseal Key 5: base64_encoded_key_5

Initial Root Token: hvs.xxxxxxxxxxxxxxxxxxxxx
```

### Unsealing Vault

```bash
# Vault требует 3 ключа из 5 для unsealing
vault operator unseal <Unseal_Key_1>
vault operator unseal <Unseal_Key_2>
vault operator unseal <Unseal_Key_3>

# Проверка статуса
vault status
# Sealed: false означает успешный unseal

exit
```

### Автоматический unseal при рестарте

Создайте Kubernetes Secret с unseal ключами:

```bash
# ВНИМАНИЕ: Это упрощенный подход для dev/test
# В production используйте Auto-unseal с KMS

kubectl -n vault create secret generic vault-unseal-keys \
  --from-literal=key1='<Unseal_Key_1>' \
  --from-literal=key2='<Unseal_Key_2>' \
  --from-literal=key3='<Unseal_Key_3>'
```

Создайте init container для автоматического unseal:

```bash
cat > /tmp/vault-unseal-job.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: vault-unseal
  namespace: vault
spec:
  template:
    spec:
      serviceAccountName: vault
      containers:
      - name: vault-unseal
        image: hashicorp/vault:1.15.4
        env:
        - name: VAULT_ADDR
          value: "http://vault:8200"
        - name: UNSEAL_KEY_1
          valueFrom:
            secretKeyRef:
              name: vault-unseal-keys
              key: key1
        - name: UNSEAL_KEY_2
          valueFrom:
            secretKeyRef:
              name: vault-unseal-keys
              key: key2
        - name: UNSEAL_KEY_3
          valueFrom:
            secretKeyRef:
              name: vault-unseal-keys
              key: key3
        command:
        - /bin/sh
        - -c
        - |
          vault operator unseal $UNSEAL_KEY_1
          vault operator unseal $UNSEAL_KEY_2
          vault operator unseal $UNSEAL_KEY_3
      restartPolicy: OnFailure
EOF

# Применить при необходимости
# kubectl apply -f /tmp/vault-unseal-job.yaml
```

### Вход в Vault CLI

```bash
# Установка Vault CLI на jumphost (если не установлен)
wget https://releases.hashicorp.com/vault/1.15.4/vault_1.15.4_linux_amd64.zip
unzip vault_1.15.4_linux_amd64.zip
sudo mv vault /usr/local/bin/
vault version

# Настройка переменных окружения
export VAULT_ADDR='http://vault.local.lab'
export VAULT_TOKEN='<Initial_Root_Token>'

# Проверка подключения
vault status
vault secrets list
```

### Доступ к Web UI

Откройте браузер: `http://vault.local.lab`

- **Method**: Token
- **Token**: `<Initial_Root_Token>`

---

## Настройка PKI Engine

### Включение PKI Secret Engine

```bash
# Root CA (для подписи intermediate CA)
vault secrets enable -path=pki pki

# Установка TTL для Root CA (10 лет)
vault secrets tune -max-lease-ttl=87600h pki

# Intermediate CA (для выдачи сертификатов)
vault secrets enable -path=pki_int pki

# Установка TTL для Intermediate CA (5 лет)
vault secrets tune -max-lease-ttl=43800h pki_int
```

### Генерация Root CA

```bash
# Генерация Root CA сертификата
vault write -field=certificate pki/root/generate/internal \
    common_name="DevOps Root CA" \
    issuer_name="root-2024" \
    ttl=87600h > /tmp/root_ca.crt

# Настройка URLs для CRL и issuing
vault write pki/config/urls \
    issuing_certificates="$VAULT_ADDR/v1/pki/ca" \
    crl_distribution_points="$VAULT_ADDR/v1/pki/crl"

# Просмотр Root CA сертификата
cat /tmp/root_ca.crt
vault read pki/cert/ca
```

### Генерация Intermediate CA

```bash
# Генерация CSR для Intermediate CA
vault write -field=csr pki_int/intermediate/generate/internal \
    common_name="DevOps Intermediate CA" \
    issuer_name="intermediate-2024" \
    ttl=43800h > /tmp/pki_intermediate.csr

# Подписание Intermediate CSR с помощью Root CA
vault write -field=certificate pki/root/sign-intermediate \
    issuer_ref="root-2024" \
    csr=@/tmp/pki_intermediate.csr \
    format=pem_bundle \
    ttl=43800h > /tmp/intermediate.cert.pem

# Импорт подписанного сертификата в Intermediate CA
vault write pki_int/intermediate/set-signed \
    certificate=@/tmp/intermediate.cert.pem

# Настройка URLs для Intermediate CA
vault write pki_int/config/urls \
    issuing_certificates="$VAULT_ADDR/v1/pki_int/ca" \
    crl_distribution_points="$VAULT_ADDR/v1/pki_int/crl"
```

### Создание роли PKI

Создайте роль для выдачи сертификатов приложениям:

```bash
# Роль для Kubernetes сервисов
vault write pki_int/roles/local-lab \
    allowed_domains="local.lab" \
    allow_subdomains=true \
    max_ttl="720h" \
    ttl="168h" \
    key_type="rsa" \
    key_bits=2048 \
    allow_any_name=false \
    allow_localhost=false \
    client_flag=true \
    server_flag=true

# Роль для wildcard сертификатов
vault write pki_int/roles/wildcard-local-lab \
    allowed_domains="local.lab" \
    allow_subdomains=true \
    allow_wildcard_certificates=true \
    max_ttl="2160h" \
    ttl="720h" \
    key_type="rsa" \
    key_bits=2048

# Роль для mTLS между сервисами
vault write pki_int/roles/mtls-services \
    allowed_domains="*.local.lab,*.svc.cluster.local" \
    allow_subdomains=true \
    max_ttl="720h" \
    ttl="168h" \
    key_type="ec" \
    key_bits=256 \
    client_flag=true \
    server_flag=true \
    code_signing_flag=false

# Просмотр ролей
vault list pki_int/roles
vault read pki_int/roles/local-lab
```

### Тестовая выдача сертификата

```bash
# Выдача тестового сертификата
vault write pki_int/issue/local-lab \
    common_name="test.local.lab" \
    ttl="24h"

# Сохранение сертификата и ключа
vault write -format=json pki_int/issue/local-lab \
    common_name="test.local.lab" \
    alt_names="www.test.local.lab" \
    ttl="24h" | jq -r '.data.certificate' > /tmp/test.crt

vault write -format=json pki_int/issue/local-lab \
    common_name="test.local.lab" \
    alt_names="www.test.local.lab" \
    ttl="24h" | jq -r '.data.private_key' > /tmp/test.key

# Проверка сертификата
openssl x509 -in /tmp/test.crt -text -noout
```

---

## Интеграция с cert-manager

### Установка cert-manager

```bash
# Установка cert-manager через Helm
helm repo add jetstack https://charts.jetstack.io
helm repo update

kubectl create namespace cert-manager

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.13.3 \
  --set installCRDs=true \
  --set prometheus.enabled=true

# Ожидание готовности
kubectl -n cert-manager wait --for=condition=ready pod \
  --all --timeout=300s

# Проверка
kubectl -n cert-manager get pods
```

### Настройка AppRole для cert-manager

cert-manager будет использовать AppRole для аутентификации в Vault:

```bash
# Включение AppRole auth
vault auth enable approle

# Создание политики для cert-manager
vault policy write cert-manager-policy - <<EOF
path "pki_int/sign/local-lab" {
  capabilities = ["create", "update"]
}

path "pki_int/sign/wildcard-local-lab" {
  capabilities = ["create", "update"]
}

path "pki_int/sign/mtls-services" {
  capabilities = ["create", "update"]
}

path "pki_int/issue/local-lab" {
  capabilities = ["create", "update"]
}

path "pki_int/issue/wildcard-local-lab" {
  capabilities = ["create", "update"]
}

path "pki_int/issue/mtls-services" {
  capabilities = ["create", "update"]
}
EOF

# Создание AppRole
vault write auth/approle/role/cert-manager \
    token_ttl=1h \
    token_max_ttl=4h \
    policies="cert-manager-policy" \
    bind_secret_id=true \
    secret_id_ttl=0

# Получение Role ID
vault read auth/approle/role/cert-manager/role-id
# Сохраните: role_id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx

# Генерация Secret ID
vault write -f auth/approle/role/cert-manager/secret-id
# Сохраните: secret_id: yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
```

### Создание Secret в Kubernetes

```bash
# Сохраните Role ID и Secret ID в переменных
ROLE_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
SECRET_ID="yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"

# Создание Secret
kubectl -n cert-manager create secret generic vault-approle \
  --from-literal=roleId="$ROLE_ID" \
  --from-literal=secretId="$SECRET_ID"

# Проверка
kubectl -n cert-manager get secret vault-approle
```

### Создание Vault Issuer

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-issuer
spec:
  vault:
    path: pki_int/sign/local-lab
    server: http://vault.vault.svc.cluster.local:8200
    auth:
      appRole:
        path: approle
        roleId: "$ROLE_ID"
        secretRef:
          name: vault-approle
          key: secretId
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-issuer-wildcard
spec:
  vault:
    path: pki_int/sign/wildcard-local-lab
    server: http://vault.vault.svc.cluster.local:8200
    auth:
      appRole:
        path: approle
        roleId: "$ROLE_ID"
        secretRef:
          name: vault-approle
          key: secretId
EOF

# Проверка ClusterIssuers
kubectl get clusterissuer
kubectl describe clusterissuer vault-issuer
```

---

## Автоматическая выдача сертификатов

### Тестовый сертификат через Certificate CRD

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: test-certificate
  namespace: default
spec:
  secretName: test-tls-secret
  duration: 720h  # 30 дней
  renewBefore: 168h  # Обновить за 7 дней до истечения
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: test.local.lab
  dnsNames:
  - test.local.lab
  - www.test.local.lab
EOF

# Ожидание выдачи
kubectl wait --for=condition=ready certificate/test-certificate --timeout=60s

# Проверка
kubectl get certificate
kubectl describe certificate test-certificate
kubectl get secret test-tls-secret
```

### Просмотр выданного сертификата

```bash
# Извлечение сертификата
kubectl get secret test-tls-secret -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/cert.crt

# Просмотр деталей
openssl x509 -in /tmp/cert.crt -text -noout | grep -A2 "Subject:"
openssl x509 -in /tmp/cert.crt -text -noout | grep -A2 "Validity"
openssl x509 -in /tmp/cert.crt -text -noout | grep -A5 "Subject Alternative Name"
```

### Автоматический TLS для Ingress

Настройка Traefik Ingress с автоматической выдачей сертификатов:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress-with-tls
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: "vault-issuer"
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  tls:
  - hosts:
    - myapp.local.lab
    secretName: myapp-tls
  rules:
  - host: myapp.local.lab
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-service
            port:
              number: 80
EOF
```

### Wildcard сертификат

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-local-lab
  namespace: default
spec:
  secretName: wildcard-tls-secret
  duration: 2160h  # 90 дней
  renewBefore: 720h  # Обновить за 30 дней
  issuerRef:
    name: vault-issuer-wildcard
    kind: ClusterIssuer
  commonName: "*.local.lab"
  dnsNames:
  - "*.local.lab"
  - local.lab
EOF

# Проверка
kubectl get certificate wildcard-local-lab
kubectl describe certificate wildcard-local-lab
```

### Применение сертификатов к существующим сервисам

#### ArgoCD с TLS

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: argocd-tls
  namespace: argocd
spec:
  secretName: argocd-server-tls
  duration: 720h
  renewBefore: 168h
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: argocd.local.lab
  dnsNames:
  - argocd.local.lab
EOF

# Обновление Ingress для ArgoCD
kubectl -n argocd patch ingress argocd-server-ingress -p '
{
  "metadata": {
    "annotations": {
      "traefik.ingress.kubernetes.io/router.tls": "true"
    }
  },
  "spec": {
    "tls": [{
      "hosts": ["argocd.local.lab"],
      "secretName": "argocd-server-tls"
    }]
  }
}'
```

#### Grafana с TLS

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: grafana-tls
  namespace: monitoring
spec:
  secretName: grafana-server-tls
  duration: 720h
  renewBefore: 168h
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: grafana.local.lab
  dnsNames:
  - grafana.local.lab
EOF

# Обновление Ingress
kubectl -n monitoring patch ingress grafana-ingress -p '
{
  "metadata": {
    "annotations": {
      "traefik.ingress.kubernetes.io/router.tls": "true"
    }
  },
  "spec": {
    "tls": [{
      "hosts": ["grafana.local.lab"],
      "secretName": "grafana-server-tls"
    }]
  }
}'
```

#### EasyShop с TLS

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: easyshop-tls
  namespace: easyshop
spec:
  secretName: easyshop-tls-secret
  duration: 720h
  renewBefore: 168h
  issuerRef:
    name: vault-issuer
    kind: ClusterIssuer
  commonName: easyshop.local.lab
  dnsNames:
  - easyshop.local.lab
  - www.easyshop.local.lab
EOF

# Обновление Ingress
kubectl -n easyshop patch ingress easyshop-ingress -p '
{
  "metadata": {
    "annotations": {
      "traefik.ingress.kubernetes.io/router.tls": "true"
    }
  },
  "spec": {
    "tls": [{
      "hosts": ["easyshop.local.lab"],
      "secretName": "easyshop-tls-secret"
    }]
  }
}'
```

---

## Политики доступа

### Создание политик для приложений

```bash
# Политика для чтения сертификатов
vault policy write app-read-certs - <<EOF
# Разрешить чтение выданных сертификатов
path "pki_int/cert/*" {
  capabilities = ["read"]
}

# Разрешить просмотр CA
path "pki_int/ca" {
  capabilities = ["read"]
}

# Разрешить просмотр CRL
path "pki_int/crl" {
  capabilities = ["read"]
}
EOF

# Политика для администраторов PKI
vault policy write pki-admin - <<EOF
# Полный доступ к PKI
path "pki_int/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Управление ролями
path "pki_int/roles/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Управление конфигурацией
path "pki_int/config/*" {
  capabilities = ["create", "read", "update", "delete"]
}

# Просмотр выданных сертификатов
path "pki_int/certs" {
  capabilities = ["list"]
}

# Отзыв сертификатов
path "pki_int/revoke" {
  capabilities = ["create", "update"]
}
EOF

# Политика для мониторинга
vault policy write monitoring - <<EOF
# Доступ к метрикам
path "sys/metrics" {
  capabilities = ["read"]
}

# Доступ к health check
path "sys/health" {
  capabilities = ["read"]
}

# Доступ к audit logs
path "sys/audit" {
  capabilities = ["read"]
}
EOF
```

### Kubernetes Auth Method

Настройка аутентификации для приложений в Kubernetes:

```bash
# Включение Kubernetes auth
vault auth enable kubernetes

# Настройка Kubernetes auth
vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

# Создание роли для приложений
vault write auth/kubernetes/role/easyshop \
    bound_service_account_names=easyshop \
    bound_service_account_namespaces=easyshop \
    policies=app-read-certs \
    ttl=24h

# Создание ServiceAccount для приложения
kubectl -n easyshop create serviceaccount easyshop
```

---

## Мониторинг и логирование

### Включение аудит логов

```bash
# Включение файлового аудита
vault audit enable file file_path=/vault/audit/audit.log

# Проверка
vault audit list
```

### Экспорт метрик в Prometheus

```bash
# Создание ServiceMonitor для Prometheus
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: vault-metrics
  namespace: vault
  labels:
    app.kubernetes.io/name: vault
spec:
  selector:
    app.kubernetes.io/name: vault
  ports:
  - name: metrics
    port: 8200
    targetPort: 8200
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vault
  namespace: vault
  labels:
    app: vault
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: vault
  endpoints:
  - port: metrics
    path: /v1/sys/metrics
    params:
      format: ['prometheus']
    interval: 30s
EOF
```

### Дашборд Grafana для Vault

```bash
# Импортируйте дашборд в Grafana
# Dashboard ID: 12904 (HashiCorp Vault)
# Или создайте кастомный дашборд
```

### Алерты для Vault

```bash
cat <<EOF | kubectl apply -f -
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: vault-alerts
  namespace: vault
  labels:
    prometheus: kube-prometheus
spec:
  groups:
  - name: vault
    interval: 30s
    rules:
    - alert: VaultSealed
      expr: vault_core_unsealed == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Vault запечатан"
        description: "Vault экземпляр {{ \$labels.instance }} запечатан"
    
    - alert: VaultDown
      expr: up{job="vault"} == 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Vault недоступен"
        description: "Vault экземпляр {{ \$labels.instance }} недоступен"
    
    - alert: VaultCertificateExpiringSoon
      expr: vault_pki_certificate_expiry_seconds < 604800
      labels:
        severity: warning
      annotations:
        summary: "Сертификат истекает скоро"
        description: "Сертификат {{ \$labels.common_name }} истекает через {{ \$value | humanizeDuration }}"
    
    - alert: VaultHighMemoryUsage
      expr: container_memory_usage_bytes{pod=~"vault-.*"} / container_spec_memory_limit_bytes{pod=~"vault-.*"} > 0.9
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Высокое использование памяти Vault"
        description: "Pod {{ \$labels.pod }} использует {{ \$value | humanizePercentage }} памяти"
EOF
```

---

## Backup и восстановление

### Создание backup стратегии

```bash
# Создание скрипта для backup
cat > /tmp/vault-backup.sh <<'SCRIPT'
#!/bin/bash
set -e

BACKUP_DIR="/tmp/vault-backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/vault-backup-$TIMESTAMP.snap"

mkdir -p $BACKUP_DIR

# Создание snapshot (только для HA режима с Raft)
kubectl -n vault exec vault-0 -- vault operator raft snapshot save /tmp/backup.snap

# Копирование snapshot из pod
kubectl -n vault cp vault-0:/tmp/backup.snap $BACKUP_FILE

# Backup в MinIO
mc cp $BACKUP_FILE localminio/backups/vault/

# Удаление старых backup (хранить последние 7 дней)
find $BACKUP_DIR -name "vault-backup-*.snap" -mtime +7 -delete

echo "Backup завершен: $BACKUP_FILE"
SCRIPT

chmod +x /tmp/vault-backup.sh
```

### CronJob для автоматического backup

```bash
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-backup
  namespace: vault
spec:
  schedule: "0 2 * * *"  # Ежедневно в 2:00
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: vault
          containers:
          - name: backup
            image: hashicorp/vault:1.15.4
            command:
            - /bin/sh
            - -c
            - |
              vault operator raft snapshot save /tmp/backup.snap
              # Здесь добавьте команды для копирования в MinIO
          restartPolicy: OnFailure
EOF
```

### Восстановление из backup

```bash
# Восстановление Raft snapshot
kubectl -n vault exec vault-0 -- vault operator raft snapshot restore /path/to/backup.snap

# Или через файл
kubectl -n vault cp backup.snap vault-0:/tmp/backup.snap
kubectl -n vault exec vault-0 -- vault operator raft snapshot restore /tmp/backup.snap
```

### Backup конфигурации и секретов

```bash
# Экспорт PKI конфигурации
vault read -format=json pki_int/config/urls > /tmp/pki-config.json
vault list -format=json pki_int/roles > /tmp/pki-roles.json

# Экспорт политик
vault policy list | while read policy; do
  vault policy read $policy > /tmp/policy-$policy.hcl
done

# Экспорт auth methods
vault auth list -format=json > /tmp/auth-methods.json
```

---

## Advanced конфигурации

### Настройка Auto-Unseal с Transit Engine

Для production рекомендуется использовать Auto-Unseal:

```bash
# На отдельном Vault кластере (Transit Vault)
vault secrets enable transit
vault write -f transit/keys/autounseal

# Создание политики для auto-unseal
vault policy write autounseal - <<EOF
path "transit/encrypt/autounseal" {
   capabilities = [ "update" ]
}

path "transit/decrypt/autounseal" {
   capabilities = [ "update" ]
}
EOF

# Создание токена для auto-unseal
vault token create -policy=autounseal -orphan -period=24h
```

Обновите конфигурацию Vault для использования Auto-Unseal:

```bash
cat > /tmp/vault-autounseal-values.yaml <<'EOF'
server:
  standalone:
    config: |
      ui = true
      
      listener "tcp" {
        tls_disable = 1
        address = "[::]:8200"
        cluster_address = "[::]:8201"
      }
      
      storage "file" {
        path = "/vault/data"
      }
      
      seal "transit" {
        address = "https://transit-vault.local.lab"
        token = "hvs.xxxxx"
        disable_renewal = "false"
        key_name = "autounseal"
        mount_path = "transit/"
        tls_skip_verify = "true"
      }
EOF

# Обновление через Helm
helm upgrade vault hashicorp/vault \
  --namespace vault \
  --values /tmp/vault-autounseal-values.yaml
```

### Multi-Region PKI Setup

Настройка нескольких Intermediate CA для разных регионов:

```bash
# Создание Intermediate CA для региона EU
vault secrets enable -path=pki_eu pki
vault secrets tune -max-lease-ttl=43800h pki_eu

vault write -field=csr pki_eu/intermediate/generate/internal \
    common_name="DevOps EU Intermediate CA" \
    issuer_name="intermediate-eu-2024" \
    ttl=43800h > /tmp/pki_eu_intermediate.csr

vault write -field=certificate pki/root/sign-intermediate \
    issuer_ref="root-2024" \
    csr=@/tmp/pki_eu_intermediate.csr \
    format=pem_bundle \
    ttl=43800h > /tmp/intermediate_eu.cert.pem

vault write pki_eu/intermediate/set-signed \
    certificate=@/tmp/intermediate_eu.cert.pem

# Создание Intermediate CA для региона US
vault secrets enable -path=pki_us pki
vault secrets tune -max-lease-ttl=43800h pki_us

vault write -field=csr pki_us/intermediate/generate/internal \
    common_name="DevOps US Intermediate CA" \
    issuer_name="intermediate-us-2024" \
    ttl=43800h > /tmp/pki_us_intermediate.csr

vault write -field=certificate pki/root/sign-intermediate \
    issuer_ref="root-2024" \
    csr=@/tmp/pki_us_intermediate.csr \
    format=pem_bundle \
    ttl=43800h > /tmp/intermediate_us.cert.pem

vault write pki_us/intermediate/set-signed \
    certificate=@/tmp/intermediate_us.cert.pem
```

### Certificate Rotation Strategy

Автоматизация ротации сертификатов:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cert-rotation-script
  namespace: vault
data:
  rotate-certs.sh: |
    #!/bin/bash
    
    # Получение списка сертификатов, истекающих в течение 7 дней
    EXPIRING_CERTS=\$(vault list -format=json pki_int/certs | jq -r '.[]' | while read serial; do
      CERT_INFO=\$(vault read -format=json pki_int/cert/\$serial)
      EXPIRY=\$(echo \$CERT_INFO | jq -r '.data.certificate' | openssl x509 -noout -enddate | cut -d= -f2)
      EXPIRY_EPOCH=\$(date -d "\$EXPIRY" +%s)
      NOW_EPOCH=\$(date +%s)
      DAYS_LEFT=\$(( (\$EXPIRY_EPOCH - \$NOW_EPOCH) / 86400 ))
      
      if [ \$DAYS_LEFT -lt 7 ]; then
        CN=\$(echo \$CERT_INFO | jq -r '.data.certificate' | openssl x509 -noout -subject | sed 's/.*CN = //')
        echo "\$CN:\$serial:\$DAYS_LEFT"
      fi
    done)
    
    # Отправка уведомлений
    if [ -n "\$EXPIRING_CERTS" ]; then
      echo "Сертификаты, требующие обновления:"
      echo "\$EXPIRING_CERTS"
      # Здесь можно добавить отправку в Slack/Email
    fi
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cert-rotation-check
  namespace: vault
spec:
  schedule: "0 8 * * *"  # Ежедневно в 8:00
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: vault
          containers:
          - name: rotation-check
            image: hashicorp/vault:1.15.4
            command:
            - /bin/sh
            - /scripts/rotate-certs.sh
            volumeMounts:
            - name: scripts
              mountPath: /scripts
          volumes:
          - name: scripts
            configMap:
              name: cert-rotation-script
              defaultMode: 0755
          restartPolicy: OnFailure
EOF
```

### OCSP Responder Setup

Настройка Online Certificate Status Protocol:

```bash
# Включение OCSP в PKI
vault write pki_int/config/urls \
    issuing_certificates="$VAULT_ADDR/v1/pki_int/ca" \
    crl_distribution_points="$VAULT_ADDR/v1/pki_int/crl" \
    ocsp_servers="$VAULT_ADDR/v1/pki_int/ocsp"

# Проверка OCSP
CERT_SERIAL=$(openssl x509 -in /tmp/test.crt -noout -serial | cut -d= -f2)
curl -v "$VAULT_ADDR/v1/pki_int/ocsp/$CERT_SERIAL"
```

---

## Production Deployment

### High Availability Configuration

Полная HA конфигурация для production:

```bash
cat > /tmp/vault-ha-values.yaml <<'EOF'
global:
  enabled: true
  tlsDisable: false

server:
  image:
    repository: hashicorp/vault
    tag: "1.15.4"
  
  # HA режим с Raft
  ha:
    enabled: true
    replicas: 5
    raft:
      enabled: true
      setNodeId: true
      
      config: |
        ui = true
        
        listener "tcp" {
          address = "[::]:8200"
          cluster_address = "[::]:8201"
          tls_cert_file = "/vault/tls/tls.crt"
          tls_key_file = "/vault/tls/tls.key"
          tls_client_ca_file = "/vault/tls/ca.crt"
        }
        
        storage "raft" {
          path = "/vault/data"
          
          retry_join {
            leader_api_addr = "https://vault-0.vault-internal:8200"
            leader_ca_cert_file = "/vault/tls/ca.crt"
            leader_client_cert_file = "/vault/tls/tls.crt"
            leader_client_key_file = "/vault/tls/tls.key"
          }
          
          retry_join {
            leader_api_addr = "https://vault-1.vault-internal:8200"
            leader_ca_cert_file = "/vault/tls/ca.crt"
            leader_client_cert_file = "/vault/tls/tls.crt"
            leader_client_key_file = "/vault/tls/tls.key"
          }
          
          retry_join {
            leader_api_addr = "https://vault-2.vault-internal:8200"
            leader_ca_cert_file = "/vault/tls/ca.crt"
            leader_client_cert_file = "/vault/tls/tls.crt"
            leader_client_key_file = "/vault/tls/tls.key"
          }
        }
        
        service_registration "kubernetes" {}
        
        seal "transit" {
          address = "https://transit-vault.local.lab"
          disable_renewal = "false"
          key_name = "autounseal"
          mount_path = "transit/"
        }
        
        telemetry {
          prometheus_retention_time = "30s"
          disable_hostname = true
        }
  
  # Ресурсы для production
  resources:
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 2Gi
      cpu: 1000m
  
  # Affinity для распределения по нодам
  affinity: |
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: vault
              app.kubernetes.io/instance: vault
          topologyKey: kubernetes.io/hostname
  
  # Storage
  dataStorage:
    enabled: true
    size: 50Gi
    storageClass: longhorn
    accessMode: ReadWriteOnce
  
  auditStorage:
    enabled: true
    size: 20Gi
    storageClass: longhorn

  # Service
  service:
    enabled: true
    type: LoadBalancer
    port: 8200

ui:
  enabled: true
  serviceType: LoadBalancer

injector:
  enabled: true
  replicas: 2
  
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: 500m
  
  affinity: |
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: vault-agent-injector
              app.kubernetes.io/instance: vault
          topologyKey: kubernetes.io/hostname
EOF

# Установка HA Vault
helm upgrade --install vault hashicorp/vault \
  --namespace vault \
  --values /tmp/vault-ha-values.yaml
```

### Resource Limits и Quotas

```bash
# Создание ResourceQuota для namespace vault
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: vault-quota
  namespace: vault
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: vault-limit-range
  namespace: vault
spec:
  limits:
  - max:
      cpu: "2"
      memory: 4Gi
    min:
      cpu: 100m
      memory: 128Mi
    default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 250m
      memory: 256Mi
    type: Container
EOF
```

### Network Policies

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: vault-network-policy
  namespace: vault
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: vault
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Доступ от cert-manager
  - from:
    - namespaceSelector:
        matchLabels:
          name: cert-manager
    ports:
    - protocol: TCP
      port: 8200
  # Доступ от приложений
  - from:
    - namespaceSelector:
        matchLabels:
          vault-access: "true"
    ports:
    - protocol: TCP
      port: 8200
  # Raft communication между Vault pods
  - from:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: vault
    ports:
    - protocol: TCP
      port: 8201
  egress:
  # DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Raft communication
  - to:
    - podSelector:
        matchLabels:
          app.kubernetes.io/name: vault
    ports:
    - protocol: TCP
      port: 8200
    - protocol: TCP
      port: 8201
EOF
```

### Pod Security Policies

```bash
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: vault-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  supplementalGroups:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  readOnlyRootFilesystem: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: vault-psp-role
  namespace: vault
rules:
- apiGroups: ['policy']
  resources: ['podsecuritypolicies']
  verbs: ['use']
  resourceNames: ['vault-psp']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: vault-psp-rolebinding
  namespace: vault
roleRef:
  kind: Role
  name: vault-psp-role
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: vault
  namespace: vault
EOF
```

---

## Disaster Recovery

### Backup Automation с Velero

```bash
# Установка Velero
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update

kubectl create namespace velero

# Создание backup location в MinIO
mc mb localminio/velero-backups

# Создание credentials для Velero
cat > /tmp/credentials-velero <<EOF
[default]
aws_access_key_id = minioadmin
aws_secret_access_key = minioadmin
EOF

# Установка Velero
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --set-file credentials.secretContents.cloud=/tmp/credentials-velero \
  --set configuration.provider=aws \
  --set configuration.backupStorageLocation.bucket=velero-backups \
  --set configuration.backupStorageLocation.config.region=us-east-1 \
  --set configuration.backupStorageLocation.config.s3ForcePathStyle=true \
  --set configuration.backupStorageLocation.config.s3Url=http://minio.minio.svc.cluster.local:9000 \
  --set snapshotsEnabled=true \
  --set deployRestic=true

# Создание backup schedule для Vault
cat <<EOF | kubectl apply -f -
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: vault-backup
  namespace: velero
spec:
  schedule: "0 2 * * *"  # Ежедневно в 2:00
  template:
    includedNamespaces:
    - vault
    includedResources:
    - '*'
    storageLocation: default
    ttl: 168h0m0s  # 7 дней
EOF
```

### DR Runbook

Документация процедур disaster recovery:

```bash
cat > /tmp/vault-dr-runbook.md <<'EOF'
# Vault Disaster Recovery Runbook

## Сценарий 1: Потеря одного Vault pod в HA

1. Проверка статуса кластера:
   ```bash
   kubectl -n vault get pods
   kubectl -n vault exec vault-0 -- vault operator raft list-peers
   ```

2. Pod автоматически пересоздастся и присоединится к кластеру
3. Проверка после восстановления:
   ```bash
   kubectl -n vault logs vault-X
   kubectl -n vault exec vault-0 -- vault status
   ```

## Сценарий 2: Полная потеря Vault кластера

1. Восстановление из Velero backup:
   ```bash
   velero restore create --from-backup vault-backup-YYYYMMDD
   ```

2. Unsealing всех pods (если не настроен Auto-unseal):
   ```bash
   for i in 0 1 2; do
     kubectl -n vault exec vault-$i -- vault operator unseal <key1>
     kubectl -n vault exec vault-$i -- vault operator unseal <key2>
     kubectl -n vault exec vault-$i -- vault operator unseal <key3>
   done
   ```

3. Проверка Raft кластера:
   ```bash
   kubectl -n vault exec vault-0 -- vault operator raft list-peers
   ```

## Сценарий 3: Corruption данных Raft

1. Остановка всех Vault pods кроме leader:
   ```bash
   kubectl -n vault scale statefulset vault --replicas=1
   ```

2. Восстановление из snapshot:
   ```bash
   kubectl -n vault cp backup.snap vault-0:/tmp/backup.snap
   kubectl -n vault exec vault-0 -- vault operator raft snapshot restore /tmp/backup.snap
   ```

3. Масштабирование обратно:
   ```bash
   kubectl -n vault scale statefulset vault --replicas=5
   ```

## Сценарий 4: Потеря Root CA

1. Восстановление Root CA из backup:
   ```bash
   vault write pki/config/ca pem_bundle=@/backup/root-ca-bundle.pem
   ```

2. Пересоздание Intermediate CA:
   ```bash
   # Следуйте процедуре из раздела "Настройка PKI Engine"
   ```

3. Обновление всех ClusterIssuers в cert-manager

## Контакты для escalation

- DevOps Team: devops@company.com
- Security Team: security@company.com
- On-call: +7-XXX-XXX-XXXX
EOF
```

### Тестирование DR процедур

```bash
# Скрипт для регулярного тестирования DR
cat > /tmp/test-dr.sh <<'SCRIPT'
#!/bin/bash

echo "=== Тестирование Disaster Recovery ==="

# 1. Создание тестового backup
echo "[1/5] Создание backup..."
velero backup create vault-dr-test --include-namespaces vault --wait

# 2. Удаление тестового namespace
echo "[2/5] Создание test namespace..."
kubectl create namespace vault-dr-test

# 3. Восстановление в test namespace
echo "[3/5] Восстановление из backup..."
velero restore create vault-dr-restore-test \
  --from-backup vault-dr-test \
  --namespace-mappings vault:vault-dr-test \
  --wait

# 4. Проверка восстановленных ресурсов
echo "[4/5] Проверка ресурсов..."
kubectl -n vault-dr-test get all

# 5. Cleanup
echo "[5/5] Очистка..."
kubectl delete namespace vault-dr-test
velero backup delete vault-dr-test --confirm

echo "=== DR тест завершен ==="
SCRIPT

chmod +x /tmp/test-dr.sh
```

---

## Интеграция с внешними системами

### LDAP Authentication

```bash
# Включение LDAP auth
vault auth enable ldap

# Настройка LDAP
vault write auth/ldap/config \
    url="ldap://ldap.company.com" \
    userdn="ou=users,dc=company,dc=com" \
    groupdn="ou=groups,dc=company,dc=com" \
    binddn="cn=vault,ou=service-accounts,dc=company,dc=com" \
    bindpass="password" \
    userattr="uid" \
    groupattr="cn"

# Mapping LDAP групп на Vault политики
vault write auth/ldap/groups/devops policies=pki-admin
vault write auth/ldap/groups/developers policies=app-read-certs
vault write auth/ldap/groups/monitoring policies=monitoring

# Тест LDAP auth
vault login -method=ldap username=john.doe
```

### Slack Notifications

```bash
cat > /tmp/vault-slack-webhook.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-notifications
  namespace: vault
data:
  send-notification.sh: |
    #!/bin/bash
    
    SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
    
    send_slack() {
      local message="$1"
      local color="$2"
      
      curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-Type: application/json' \
        -d "{
          \"attachments\": [{
            \"color\": \"$color\",
            \"title\": \"Vault Alert\",
            \"text\": \"$message\",
            \"footer\": \"Vault Monitoring\",
            \"ts\": $(date +%s)
          }]
        }"
    }
    
    # Проверка sealed status
    if kubectl -n vault exec vault-0 -- vault status | grep -q "Sealed.*true"; then
      send_slack "⚠️ Vault is SEALED!" "danger"
    fi
    
    # Проверка истекающих сертификатов
    EXPIRING=$(vault list -format=json pki_int/certs 2>/dev/null | \
      jq -r '.[] | select(. != null)' | \
      wc -l)
    
    if [ "$EXPIRING" -gt 100 ]; then
      send_slack "⚠️ More than 100 certificates expiring soon!" "warning"
    fi
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-slack-notifications
  namespace: vault
spec:
  schedule: "*/30 * * * *"  #