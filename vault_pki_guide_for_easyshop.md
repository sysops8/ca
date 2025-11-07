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
  schedule: "*/30 * * * *"  # Каждые 30 минут
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: vault
          containers:
          - name: notifications
            image: hashicorp/vault:1.15.4
            command:
            - /bin/sh
            - /scripts/send-notification.sh
            volumeMounts:
            - name: scripts
              mountPath: /scripts
          volumes:
          - name: scripts
            configMap:
              name: vault-notifications
              defaultMode: 0755
          restartPolicy: OnFailure
EOF
```

### Integration с Jira для Audit

```bash
cat > /tmp/vault-jira-integration.py <<'PYTHON'
#!/usr/bin/env python3
import json
import requests
from datetime import datetime

JIRA_URL = "https://jira.company.com"
JIRA_USER = "vault-bot"
JIRA_TOKEN = "your-api-token"
PROJECT_KEY = "SEC"

def parse_audit_log(log_line):
    """Парсинг audit log записи"""
    try:
        log_data = json.loads(log_line)
        return log_data
    except json.JSONDecodeError:
        return None

def create_jira_ticket(event_data):
    """Создание Jira тикета для критических событий"""
    
    summary = f"Vault Security Event: {event_data.get('type', 'Unknown')}"
    description = f"""
    *Event Details*
    
    Time: {event_data.get('time', 'N/A')}
    Type: {event_data.get('type', 'N/A')}
    Path: {event_data.get('request', {}).get('path', 'N/A')}
    Client: {event_data.get('request', {}).get('client_token', 'N/A')}
    Remote Address: {event_data.get('request', {}).get('remote_address', 'N/A')}
    
    *Response*
    Status: {event_data.get('response', {}).get('status', 'N/A')}
    """
    
    payload = {
        "fields": {
            "project": {"key": PROJECT_KEY},
            "summary": summary,
            "description": description,
            "issuetype": {"name": "Security Event"},
            "priority": {"name": "High"}
        }
    }
    
    response = requests.post(
        f"{JIRA_URL}/rest/api/2/issue",
        auth=(JIRA_USER, JIRA_TOKEN),
        headers={"Content-Type": "application/json"},
        json=payload
    )
    
    if response.status_code == 201:
        ticket_key = response.json()["key"]
        print(f"Created Jira ticket: {ticket_key}")
    else:
        print(f"Failed to create ticket: {response.text}")

def monitor_audit_logs():
    """Мониторинг audit logs для критических событий"""
    
    critical_events = [
        "policy-delete",
        "auth-disable",
        "seal",
        "unseal-failed"
    ]
    
    # Чтение audit log
    with open('/vault/audit/audit.log', 'r') as f:
        for line in f:
            event = parse_audit_log(line)
            if event and event.get('type') in critical_events:
                create_jira_ticket(event)

if __name__ == "__main__":
    monitor_audit_logs()
PYTHON

chmod +x /tmp/vault-jira-integration.py
```

### Prometheus Metrics Export

Расширенная конфигурация метрик:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: vault-exporter-config
  namespace: vault
data:
  prometheus.yml: |
    global:
      scrape_interval: 30s
      evaluation_interval: 30s
    
    scrape_configs:
    - job_name: 'vault'
      metrics_path: '/v1/sys/metrics'
      params:
        format: ['prometheus']
      bearer_token: 'hvs.your-monitoring-token'
      static_configs:
      - targets: ['vault.vault.svc.cluster.local:8200']
      
    - job_name: 'vault-pki'
      metrics_path: '/v1/sys/metrics'
      params:
        format: ['prometheus']
      bearer_token: 'hvs.your-monitoring-token'
      static_configs:
      - targets: ['vault.vault.svc.cluster.local:8200']
      metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'vault_pki.*'
        action: keep
EOF
```

### Custom Grafana Dashboard

```json
cat > /tmp/vault-dashboard.json <<'JSON'
{
  "dashboard": {
    "title": "Vault PKI Monitoring",
    "panels": [
      {
        "title": "Certificate Issuance Rate",
        "targets": [
          {
            "expr": "rate(vault_pki_certificate_issue_count[5m])"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Certificates Expiring Soon",
        "targets": [
          {
            "expr": "vault_pki_certificate_expiry_seconds < 604800"
          }
        ],
        "type": "stat"
      },
      {
        "title": "Active Certificates",
        "targets": [
          {
            "expr": "vault_pki_certificate_active_count"
          }
        ],
        "type": "gauge"
      },
      {
        "title": "Vault Sealed Status",
        "targets": [
          {
            "expr": "vault_core_unsealed"
          }
        ],
        "type": "stat",
        "thresholds": [
          {
            "value": 0,
            "color": "red"
          },
          {
            "value": 1,
            "color": "green"
          }
        ]
      },
      {
        "title": "Request Rate by Path",
        "targets": [
          {
            "expr": "rate(vault_route_rollback_attempt_count[5m])"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Authentication Failures",
        "targets": [
          {
            "expr": "rate(vault_token_create_failure_count[5m])"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
JSON
```

---

## Интеграция с приложениями

### Vault Agent Injector для приложений

Полный пример внедрения сертификатов в приложение:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-cert.pem: "pki_int/issue/local-lab"
        vault.hashicorp.com/agent-inject-template-cert.pem: |
          {{- with secret "pki_int/issue/local-lab" "common_name=myapp.local.lab" "ttl=24h" -}}
          {{ .Data.certificate }}
          {{ .Data.ca_chain }}
          {{- end }}
        vault.hashicorp.com/agent-inject-secret-key.pem: "pki_int/issue/local-lab"
        vault.hashicorp.com/agent-inject-template-key.pem: |
          {{- with secret "pki_int/issue/local-lab" "common_name=myapp.local.lab" "ttl=24h" -}}
          {{ .Data.private_key }}
          {{- end }}
        vault.hashicorp.com/agent-inject-secret-ca.pem: "pki_int/cert/ca"
        vault.hashicorp.com/agent-inject-template-ca.pem: |
          {{- with secret "pki_int/cert/ca" -}}
          {{ .Data.certificate }}
          {{- end }}
    spec:
      serviceAccountName: myapp
      containers:
      - name: myapp
        image: nginx:alpine
        ports:
        - containerPort: 443
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: tls-certs
          mountPath: /etc/nginx/ssl
      volumes:
      - name: nginx-config
        configMap:
          name: myapp-nginx-config
      - name: tls-certs
        emptyDir: {}
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-nginx-config
  namespace: default
data:
  nginx.conf: |
    events {
      worker_connections 1024;
    }
    
    http {
      server {
        listen 443 ssl http2;
        server_name myapp.local.lab;
        
        ssl_certificate /vault/secrets/cert.pem;
        ssl_certificate_key /vault/secrets/key.pem;
        ssl_client_certificate /vault/secrets/ca.pem;
        
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;
        
        # mTLS (опционально)
        ssl_verify_client optional;
        
        location / {
          root /usr/share/nginx/html;
          index index.html;
          
          # Передача информации о клиентском сертификате
          proxy_set_header X-Client-Cert $ssl_client_cert;
          proxy_set_header X-Client-Verify $ssl_client_verify;
        }
        
        location /health {
          access_log off;
          return 200 "healthy\n";
        }
      }
    }
EOF
```

### Настройка Kubernetes Auth для приложения

```bash
# Создание политики для приложения
vault policy write myapp-policy - <<EOF
# Выдача сертификатов
path "pki_int/issue/local-lab" {
  capabilities = ["create", "update"]
}

# Чтение CA
path "pki_int/cert/ca" {
  capabilities = ["read"]
}

# Опционально: доступ к секретам приложения
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
EOF

# Создание Kubernetes auth роли
vault write auth/kubernetes/role/myapp \
    bound_service_account_names=myapp \
    bound_service_account_namespaces=default \
    policies=myapp-policy \
    ttl=24h
```

### Sidecar паттерн для автоматического обновления

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-with-sidecar
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp-sidecar
  template:
    metadata:
      labels:
        app: myapp-sidecar
    spec:
      serviceAccountName: myapp
      containers:
      # Основное приложение
      - name: myapp
        image: nginx:alpine
        ports:
        - containerPort: 443
        volumeMounts:
        - name: tls-certs
          mountPath: /etc/nginx/ssl
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
      
      # Sidecar для обновления сертификатов
      - name: cert-renewer
        image: hashicorp/vault:1.15.4
        env:
        - name: VAULT_ADDR
          value: "http://vault.vault.svc.cluster.local:8200"
        command:
        - /bin/sh
        - -c
        - |
          # Аутентификация через Kubernetes
          VAULT_TOKEN=$(vault write -field=token auth/kubernetes/login \
            role=myapp \
            jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token)
          
          export VAULT_TOKEN
          
          # Бесконечный цикл обновления
          while true; do
            echo "Обновление сертификатов..."
            
            # Выдача нового сертификата
            vault write -format=json pki_int/issue/local-lab \
              common_name=myapp.local.lab \
              ttl=24h > /tmp/cert.json
            
            # Извлечение данных
            cat /tmp/cert.json | jq -r '.data.certificate' > /certs/tls.crt
            cat /tmp/cert.json | jq -r '.data.private_key' > /certs/tls.key
            cat /tmp/cert.json | jq -r '.data.ca_chain' > /certs/ca.crt
            
            # Перезагрузка Nginx
            nginx -s reload 2>/dev/null || true
            
            # Ожидание 12 часов перед следующим обновлением
            sleep 43200
          done
        volumeMounts:
        - name: tls-certs
          mountPath: /certs
      
      volumes:
      - name: tls-certs
        emptyDir: {}
      - name: nginx-config
        configMap:
          name: myapp-nginx-config
EOF
```

---

## Управление жизненным циклом сертификатов

### Автоматическая очистка истекших сертификатов

```bash
cat > /tmp/vault-cert-cleanup.sh <<'SCRIPT'
#!/bin/bash
set -e

echo "=== Очистка истекших сертификатов ==="

# Получение списка всех сертификатов
CERTS=$(vault list -format=json pki_int/certs | jq -r '.[]')

EXPIRED_COUNT=0
REVOKED_COUNT=0

for SERIAL in $CERTS; do
  # Получение информации о сертификате
  CERT_DATA=$(vault read -format=json pki_int/cert/$SERIAL)
  
  if [ -z "$CERT_DATA" ]; then
    continue
  fi
  
  # Проверка срока действия
  CERT=$(echo "$CERT_DATA" | jq -r '.data.certificate')
  EXPIRY=$(echo "$CERT" | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
  
  if [ -n "$EXPIRY" ]; then
    EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s 2>/dev/null || echo "0")
    NOW_EPOCH=$(date +%s)
    
    # Если сертификат истек более 30 дней назад
    if [ $EXPIRY_EPOCH -gt 0 ] && [ $((NOW_EPOCH - EXPIRY_EPOCH)) -gt 2592000 ]; then
      echo "Удаление истекшего сертификата: $SERIAL"
      vault delete pki_int/cert/$SERIAL
      EXPIRED_COUNT=$((EXPIRED_COUNT + 1))
    fi
  fi
done

echo "Удалено истекших сертификатов: $EXPIRED_COUNT"

# Tidy операция для очистки внутренних структур
echo "Выполнение tidy операции..."
vault write pki_int/tidy \
  tidy_cert_store=true \
  tidy_revoked_certs=true \
  safety_buffer=72h

echo "=== Очистка завершена ==="
SCRIPT

chmod +x /tmp/vault-cert-cleanup.sh

# CronJob для автоматической очистки
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-cert-cleanup
  namespace: vault
spec:
  schedule: "0 3 * * 0"  # Еженедельно по воскресеньям в 3:00
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: vault
          containers:
          - name: cleanup
            image: hashicorp/vault:1.15.4
            env:
            - name: VAULT_ADDR
              value: "http://vault:8200"
            - name: VAULT_TOKEN
              valueFrom:
                secretKeyRef:
                  name: vault-token
                  key: token
            command:
            - /bin/sh
            - /scripts/vault-cert-cleanup.sh
            volumeMounts:
            - name: scripts
              mountPath: /scripts
          volumes:
          - name: scripts
            configMap:
              name: vault-cleanup-scripts
          restartPolicy: OnFailure
EOF
```

### Certificate Transparency Log Integration

```bash
cat > /tmp/vault-ct-log.py <<'PYTHON'
#!/usr/bin/env python3
"""
Интеграция с Certificate Transparency Logs
"""
import json
import base64
import requests
from cryptography import x509
from cryptography.hazmat.backends import default_backend

CT_LOG_URL = "https://ct.googleapis.com/logs/argon2021/ct/v1/add-chain"

def submit_to_ct_log(cert_pem, chain_pem):
    """Отправка сертификата в CT log"""
    
    # Парсинг сертификатов
    cert = x509.load_pem_x509_certificate(cert_pem.encode(), default_backend())
    chain = x509.load_pem_x509_certificate(chain_pem.encode(), default_backend())
    
    # Подготовка данных
    cert_der = base64.b64encode(cert.public_bytes()).decode()
    chain_der = base64.b64encode(chain.public_bytes()).decode()
    
    payload = {
        "chain": [cert_der, chain_der]
    }
    
    # Отправка в CT log
    response = requests.post(CT_LOG_URL, json=payload)
    
    if response.status_code == 200:
        sct = response.json()
        print(f"Certificate submitted to CT log")
        print(f"SCT: {sct}")
        return sct
    else:
        print(f"Failed to submit to CT log: {response.text}")
        return None

def monitor_vault_certificates():
    """Мониторинг новых сертификатов в Vault"""
    
    # Здесь должна быть логика получения новых сертификатов
    # и отправки их в CT log
    pass

if __name__ == "__main__":
    monitor_vault_certificates()
PYTHON
```

### Certificate Pinning для мобильных приложений

```bash
cat > /tmp/generate-pins.sh <<'SCRIPT'
#!/bin/bash

# Генерация certificate pins для Android/iOS

CERT_FILE="/tmp/myapp.crt"

echo "=== Генерация Certificate Pins ==="

# Извлечение публичного ключа
PUBLIC_KEY=$(openssl x509 -in $CERT_FILE -pubkey -noout | \
  openssl pkey -pubin -outform der | \
  openssl dgst -sha256 -binary | \
  base64)

echo "SHA-256 Pin: $PUBLIC_KEY"

# Android Network Security Configuration
cat > /tmp/network_security_config.xml <<EOF
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <domain-config>
        <domain includeSubdomains="true">myapp.local.lab</domain>
        <pin-set expiration="2026-01-01">
            <pin digest="SHA-256">$PUBLIC_KEY</pin>
        </pin-set>
    </domain-config>
</network-security-config>
EOF

# iOS Info.plist
cat > /tmp/Info.plist.snippet <<EOF
<key>NSAppTransportSecurity</key>
<dict>
    <key>NSPinnedDomains</key>
    <dict>
        <key>myapp.local.lab</key>
        <dict>
            <key>NSIncludesSubdomains</key>
            <true/>
            <key>NSPinnedLeafIdentities</key>
            <array>
                <dict>
                    <key>SPKI-SHA256-BASE64</key>
                    <string>$PUBLIC_KEY</string>
                </dict>
            </array>
        </dict>
    </dict>
</dict>
EOF

echo "Файлы созданы:"
echo "- network_security_config.xml (Android)"
echo "- Info.plist.snippet (iOS)"
SCRIPT

chmod +x /tmp/generate-pins.sh
```

---

## Troubleshooting

### Общие проблемы

**Vault запечатан после рестарта:**

```bash
# Проверка статуса
kubectl -n vault exec vault-0 -- vault status

# Manual unseal
kubectl -n vault exec -it vault-0 -- sh
vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>
```

**cert-manager не может получить сертификат:**

```bash
# Проверка логов cert-manager
kubectl -n cert-manager logs -l app=cert-manager

# Проверка Certificate resource
kubectl describe certificate <cert-name> -n <namespace>

# Проверка CertificateRequest
kubectl get certificaterequest -n <namespace>
kubectl describe certificaterequest <request-name> -n <namespace>

# Тест подключения к Vault
kubectl -n cert-manager run test --image=curlimages/curl --rm -it -- \
  curl -v http://vault.vault.svc.cluster.local:8200/v1/sys/health
```

**AppRole аутентификация не работает:**

```bash
# Проверка AppRole
vault read auth/approle/role/cert-manager

# Тест аутентификации
vault write auth/approle/login \
  role_id="<role-id>" \
  secret_id="<secret-id>"

# Проверка Secret в Kubernetes
kubectl -n cert-manager get secret vault-approle -o yaml
```

**Сертификаты не обновляются автоматически:**

```bash
# Проверка настроек Certificate
kubectl get certificate <cert-name> -n <namespace> -o yaml

# Проверка событий
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Проверка cert-manager webhook
kubectl -n cert-manager logs -l app=webhook
```

### Расширенная диагностика

```bash
cat > /tmp/vault-diagnostics.sh <<'SCRIPT'
#!/bin/bash

echo "=== Vault Diagnostics ==="
echo ""

echo "[1] Vault Status"
kubectl -n vault exec vault-0 -- vault status || echo "Failed to get status"
echo ""

echo "[2] Vault Pods"
kubectl -n vault get pods -o wide
echo ""

echo "[3] Vault Logs (последние 20 строк)"
kubectl -n vault logs vault-0 --tail=20
echo ""

echo "[4] Storage Status"
kubectl -n vault get pvc
echo ""

echo "[5] PKI Status"
kubectl -n vault exec vault-0 -- vault list pki_int/roles || echo "Failed to list roles"
echo ""

echo "[6] Certificate Count"
CERT_COUNT=$(kubectl -n vault exec vault-0 -- vault list -format=json pki_int/certs 2>/dev/null | jq '. | length')
echo "Total certificates: ${CERT_COUNT:-0}"
echo ""

echo "[7] Recent Certificate Issues"
kubectl get events -n vault --field-selector reason=Issued --sort-by='.lastTimestamp' | tail -10
echo ""

echo "[8] cert-manager Status"
kubectl -n cert-manager get pods
echo ""

echo "[9] Failed Certificates"
kubectl get certificate -A | grep -i false || echo "No failed certificates"
echo ""

echo "[10] Vault Metrics"
curl -s http://vault.vault.svc.cluster.local:8200/v1/sys/metrics?format=prometheus | grep vault_core
echo ""

echo "=== Diagnostics Complete ==="
SCRIPT

chmod +x /tmp/vault-diagnostics.sh
```

### Debug режим для cert-manager

```bash
# Включение debug логов
kubectl -n cert-manager set env deployment/cert-manager --containers=cert-manager \
  LOG_LEVEL=debug

# Просмотр детальных логов
kubectl -n cert-manager logs -l app=cert-manager -f --tail=100

# Отключение debug режима
kubectl -n cert-manager set env deployment/cert-manager --containers=cert-manager \
  LOG_LEVEL=info
```

### Performance Troubleshooting

```bash
# Проверка производительности Vault
cat > /tmp/vault-perf-test.sh <<'SCRIPT'
#!/bin/bash

echo "=== Vault Performance Test ==="

# Тест выдачи сертификатов
ITERATIONS=10
START=$(date +%s)

for i in $(seq 1 $ITERATIONS); do
  vault write pki_int/issue/local-lab \
    common_name="test-$i.local.lab" \
    ttl=24h > /dev/null
done

END=$(date +%s)
DURATION=$((END - START))
AVG=$((DURATION / ITERATIONS))

echo "Issued $ITERATIONS certificates in ${DURATION}s"
echo "Average time per certificate: ${AVG}s"

# Проверка ресурсов
echo ""
echo "Resource usage:"
kubectl -n vault top pods
SCRIPT

chmod +x /tmp/vault-perf-test.sh
```

---

## Best Practices

### Безопасность

1. **Root Token**
   - Сразу после инициализации отзовите root token
   - Используйте политики вместо root доступа

```bash
# Отзыв root token
vault token revoke <root-token>

# Создание emergency token при необходимости
vault operator generate-root -init
```

2. **Unseal Keys**
   - Храните unseal ключи в разных безопасных местах (password managers, hardware tokens)
   - Используйте Auto-unseal для production (AWS KMS, Azure Key Vault, GCP KMS)
   - Никогда не храните ключи в Git или plain text
   - Используйте Shamir's Secret Sharing (5 ключей, порог 3)

3. **Политики доступа**
   - Применяйте принцип наименьших привилегий
   - Создавайте отдельные политики для каждого приложения/команды
   - Регулярно аудируйте и обновляйте политики
   - Используйте path-based permissions

```bash
# Пример минимальной политики
vault policy write minimal - <<EOF
# Только чтение собственных сертификатов
path "pki_int/issue/myapp" {
  capabilities = ["create"]
}
path "pki_int/cert/ca" {
  capabilities = ["read"]
}
EOF
```

4. **Rotation**
   - Регулярно меняйте AppRole Secret IDs (каждые 90 дней)
   - Используйте короткие TTL для токенов (24h или меньше)
   - Автоматизируйте rotation credentials
   - Настройте оповещения об истечении

### PKI Management

1. **Root CA**
   - Храните Root CA offline или в отдельном изолированном Vault
   - Используйте длительный TTL (10+ лет)
   - Создайте резервные копии в нескольких безопасных местах
   - Документируйте процесс восстановления
   - Используйте HSM для хранения ключей в production

2. **Intermediate CA**
   - Используйте меньший TTL (1-5 лет)
   - Создавайте отдельные Intermediate CA для разных окружений (dev, staging, prod)
   - Регулярно обновляйте перед истечением
   - Документируйте цепочку сертификатов

3. **Сертификаты приложений**
   - Используйте короткие TTL (24h-30d)
   - Включайте автоматическое обновление (renewBefore: 1/3 от TTL)
   - Настройте мониторинг истечения
   - Используйте wildcard сертификаты с осторожностью

```bash
# Рекомендуемые TTL
# Root CA: 87600h (10 лет)
# Intermediate CA: 43800h (5 лет)
# Server certificates: 720h (30 дней)
# Client certificates: 168h (7 дней)
# Development: 24h (1 день)
```

### Операционные практики

1. **Мониторинг**
   - Интегрируйте с Prometheus для метрик
   - Создайте дашборды в Grafana для визуализации
   - Настройте алерты для критических событий
   - Мониторьте использование ресурсов

2. **Аудит**
   - Включите audit logging на всех Vault серверах
   - Регулярно анализируйте логи (автоматизируйте с помощью SIEM)
   - Храните audit logs в отдельном защищенном хранилище
   - Настройте retention policy (минимум 1 год для compliance)

3. **Backup**
   - Ежедневные автоматические backup (2:00 AM)
   - Храните backup в нескольких локациях (on-site + off-site)
   - Тестируйте восстановление ежемесячно
   - Документируйте процедуры восстановления
   - Используйте encrypted backup

4. **High Availability**
   - Минимум 3 узла для production (рекомендуется 5)
   - Распределяйте узлы по availability zones
   - Настройте автоматический failover
   - Регулярно тестируйте DR процедуры

5. **Change Management**
   - Используйте GitOps для управления конфигурацией
   - Версионируйте все изменения
   - Требуйте peer review для изменений политик
   - Тестируйте изменения в non-production окружении
   - Документируйте все критические изменения

### Compliance и Governance

1. **Access Control**
   - Используйте MFA для административного доступа
   - Внедрите Just-In-Time (JIT) access для привилегированных операций
   - Логируйте все административные действия
   - Регулярно проверяйте access logs

2. **Certificate Lifecycle**
   - Определите и документируйте certificate policies
   - Автоматизируйте выдачу и обновление
   - Настройте процесс для отзыва компрометированных сертификатов
   - Ведите реестр всех выданных сертификатов

3. **Data Protection**
   - Шифруйте данные at rest и in transit
   - Используйте secure storage (Longhorn с encryption)
   - Регулярно тестируйте восстановление
   - Соблюдайте data retention policies

---

## Заключение

Вы успешно развернули и настроили:

- ✅ **HashiCorp Vault** как корпоративный центр сертификации
- ✅ **PKI Infrastructure** с Root и Intermediate CA
- ✅ **cert-manager** для автоматической выдачи сертификатов
- ✅ **TLS для всех сервисов** (ArgoCD, Grafana, EasyShop)
- ✅ **Политики доступа** и аутентификацию
- ✅ **Мониторинг и аудит** Vault операций
- ✅ **Backup стратегию** для защиты данных
- ✅ **High Availability** конфигурацию для production
- ✅ **Disaster Recovery** процедуры
- ✅ **Интеграцию с внешними системами** (LDAP, Slack, Jira)

### Карта доступа к сервисам

| Сервис | URL | Учетные данные | Назначение |
|--------|-----|----------------|------------|
| Vault UI | http://vault.local.lab | Token authentication | Управление PKI и секретами |
| EasyShop | https://easyshop.local.lab | - | E-commerce приложение с TLS |
| ArgoCD | https://argocd.local.lab | admin / password | GitOps с защищенным соединением |
| Grafana | https://grafana.local.lab | admin / admin123 | Мониторинг Vault метрик |
| Prometheus | https://prometheus.local.lab | - | Сбор метрик |

### Ключевые метрики для мониторинга

```bash
# Проверка здоровья инфраструктуры
cat > /tmp/health-check.sh <<'SCRIPT'
#!/bin/bash

echo "=== Vault PKI Health Check ==="
echo ""

# 1. Vault Status
echo "[✓] Vault Status:"
vault status | grep "Sealed" && echo "  ✓ Unsealed" || echo "  ✗ Sealed!"
echo ""

# 2. Активные сертификаты
ACTIVE_CERTS=$(vault list -format=json pki_int/certs 2>/dev/null | jq '. | length')
echo "[✓] Active Certificates: ${ACTIVE_CERTS:-0}"
echo ""

# 3. Сертификаты, истекающие в течение 7 дней
EXPIRING=$(kubectl get certificates -A -o json | \
  jq '[.items[] | select(.status.renewalTime != null)] | length')
echo "[✓] Certificates pending renewal: ${EXPIRING:-0}"
echo ""

# 4. cert-manager статус
CERTMGR_READY=$(kubectl -n cert-manager get pods -l app=cert-manager -o json | \
  jq '.items[0].status.conditions[] | select(.type=="Ready") | .status' -r)
echo "[✓] cert-manager: $CERTMGR_READY"
echo ""

# 5. Vault HA кластер
if kubectl -n vault get pods | grep -q "vault-0"; then
  RAFT_PEERS=$(kubectl -n vault exec vault-0 -- vault operator raft list-peers 2>/dev/null | grep -c "voter" || echo "N/A")
  echo "[✓] Raft cluster peers: $RAFT_PEERS"
else
  echo "[✓] Vault mode: Standalone"
fi
echo ""

# 6. Недавние ошибки
RECENT_ERRORS=$(kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp' | \
  grep -E "(vault|cert-manager)" | tail -5)
if [ -z "$RECENT_ERRORS" ]; then
  echo "[✓] No recent errors"
else
  echo "[!] Recent warnings:"
  echo "$RECENT_ERRORS"
fi
echo ""

# 7. Storage usage
STORAGE_USED=$(kubectl -n vault get pvc -o json | \
  jq '.items[] | select(.metadata.name | contains("vault")) | .status.capacity.storage' -r | head -1)
echo "[✓] Storage allocated: ${STORAGE_USED:-N/A}"
echo ""

echo "=== Health Check Complete ==="
SCRIPT

chmod +x /tmp/health-check.sh
```

### Quick Reference Commands

```bash
# === Vault Operations ===

# Проверка статуса
vault status

# Список политик
vault policy list

# Просмотр политики
vault policy read <policy-name>

# Список auth methods
vault auth list

# Список secrets engines
vault secrets list

# === PKI Operations ===

# Список ролей
vault list pki_int/roles

# Выдача сертификата
vault write pki_int/issue/local-lab \
  common_name="app.local.lab" \
  ttl=24h

# Просмотр CA
vault read pki_int/cert/ca

# Отзыв сертификата
vault write pki_int/revoke serial_number="XX:XX:XX"

# === cert-manager Operations ===

# Список сертификатов
kubectl get certificates -A

# Детали сертификата
kubectl describe certificate <cert-name> -n <namespace>

# Принудительное обновление
kubectl delete certificaterequest <request-name> -n <namespace>

# === Troubleshooting ===

# Логи Vault
kubectl -n vault logs vault-0 -f

# Логи cert-manager
kubectl -n cert-manager logs -l app=cert-manager -f

# События
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# Метрики Vault
curl http://vault.local.lab/v1/sys/metrics?format=prometheus
```

### Архитектурная диаграмма финальной системы

```
┌─────────────────────────────────────────────────────────────┐
│                     Production Environment                   │
│                                                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │              Load Balancer / Ingress               │    │
│  │                  (Traefik with TLS)                │    │
│  └───────────────────┬────────────────────────────────┘    │
│                      │                                       │
│  ┌───────────────────┴────────────────────────────────┐    │
│  │                                                     │    │
│  │  ┌──────────────────┐    ┌──────────────────┐     │    │
│  │  │   Applications   │    │   Monitoring     │     │    │
│  │  │                  │    │                  │     │    │
│  │  │ • EasyShop      │    │ • Grafana       │     │    │
│  │  │ • ArgoCD        │    │ • Prometheus    │     │    │
│  │  │ • Custom Apps   │    │ • Alertmanager  │     │    │
│  │  └────────┬─────────┘    └────────┬────────┘     │    │
│  │           │ TLS Certs             │ Metrics       │    │
│  │           └───────────┬───────────┘               │    │
│  │                       │                            │    │
│  └───────────────────────┼────────────────────────────┘    │
│                          │                                  │
│  ┌───────────────────────┴────────────────────────────┐    │
│  │              cert-manager                          │    │
│  │                                                     │    │
│  │  • Certificate CRDs                                │    │
│  │  • ClusterIssuers                                  │    │
│  │  • Automatic Renewal                               │    │
│  └───────────────────────┬────────────────────────────┘    │
│                          │ AppRole Auth                     │
│  ┌───────────────────────┴────────────────────────────┐    │
│  │          HashiCorp Vault (HA Cluster)              │    │
│  │                                                     │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐        │    │
│  │  │ vault-0  │  │ vault-1  │  │ vault-2  │        │    │
│  │  │ (Leader) │←→│ (Voter)  │←→│ (Voter)  │        │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘        │    │
│  │       │             │             │                │    │
│  │       └─────────────┴─────────────┘                │    │
│  │                     │                               │    │
│  │       ┌─────────────┴──────────────┐               │    │
│  │       │    Raft Consensus Storage   │               │    │
│  │       │      (Longhorn PVCs)        │               │    │
│  │       └──────────────────────────────┘              │    │
│  │                                                     │    │
│  │  PKI Structure:                                    │    │
│  │  • Root CA (Offline) → Intermediate CA (Online)   │    │
│  │  • Multiple Roles for different use cases         │    │
│  │  • Automated certificate issuance & rotation      │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           Infrastructure Services                    │   │
│  │                                                      │   │
│  │  • LDAP/AD Authentication                           │   │
│  │  • MinIO (Backup Storage)                           │   │
│  │  • Velero (Disaster Recovery)                       │   │
│  │  • Prometheus (Metrics)                              │   │
│  │  • Loki (Logs)                                       │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Следующие шаги для продакшена

1. **Безопасность**
   - [ ] Настройте Auto-unseal с AWS KMS / Azure Key Vault
   - [ ] Внедрите MFA для административного доступа
   - [ ] Интегрируйте с корпоративной LDAP/AD
   - [ ] Настройте Certificate Transparency logging
   - [ ] Внедрите Hardware Security Modules (HSM)

2. **High Availability**
   - [ ] Разверните минимум 5 узлов Vault
   - [ ] Распределите по availability zones
   - [ ] Настройте cross-region replication
   - [ ] Внедрите automated failover

3. **Мониторинг и Observability**
   - [ ] Создайте comprehensive Grafana dashboards
   - [ ] Настройте алерты в PagerDuty/OpsGenie
   - [ ] Интегрируйте с SIEM системой
   - [ ] Настройте distributed tracing

4. **Compliance**
   - [ ] Документируйте certificate policies
   - [ ] Настройте automated compliance checks
   - [ ] Внедрите regular security audits
   - [ ] Создайте compliance reports

5. **Automation**
   - [ ] Автоматизируйте certificate rotation
   - [ ] Внедрите ChatOps для управления
   - [ ] Создайте self-service портал
   - [ ] Настройте automated testing

6. **Disaster Recovery**
   - [ ] Документируйте DR процедуры
   - [ ] Регулярно тестируйте восстановление
   - [ ] Настройте automated backups в multiple regions
   - [ ] Создайте runbooks для всех сценариев

### Полезные ресурсы

**Документация:**
- [HashiCorp Vault Documentation](https://www.vaultproject.io/docs)
- [Vault PKI Secrets Engine](https://www.vaultproject.io/docs/secrets/pki)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)

**Обучение:**
- HashiCorp Vault Associate Certification
- Kubernetes Security Specialist (CKS)
- PKI and Certificate Management courses

**Community:**
- HashiCorp Discuss Forum
- Vault GitHub Repository
- cert-manager Slack Channel
- CNCF Security SIG

**Tools:**
- [Vault Radar](https://www.hashicorp.com/products/vault-radar) - Secret scanning
- [Boundary](https://www.boundaryproject.io/) - Secure access management
- [Consul](https://www.consul.io/) - Service mesh integration

### Контрольный список готовности к production

```bash
# Production Readiness Checklist
cat > /tmp/production-checklist.md <<'CHECKLIST'
# Vault PKI Production Readiness

## Infrastructure
- [ ] HA cluster с минимум 3 узлами
- [ ] Auto-unseal настроен и протестирован
- [ ] Persistent storage с репликацией
- [ ] Network policies настроены
- [ ] Resource limits установлены
- [ ] Pod Security Policies применены

## Security
- [ ] Root token отозван
- [ ] Unseal keys в secure storage
- [ ] TLS для всех соединений
- [ ] LDAP/AD аутентификация настроена
- [ ] MFA для административного доступа
- [ ] Audit logging включен
- [ ] Политики доступа настроены по принципу least privilege

## PKI Configuration
- [ ] Root CA создан и сохранен offline
- [ ] Intermediate CA настроен
- [ ] Certificate roles определены
- [ ] TTL политики настроены
- [ ] CRL и OCSP настроены
- [ ] Certificate Transparency включен

## Automation
- [ ] cert-manager интегрирован
- [ ] Automatic certificate renewal работает
- [ ] Certificate cleanup настроен
- [ ] Backup automation настроен
- [ ] Monitoring alerts настроены

## Monitoring
- [ ] Prometheus integration настроен
- [ ] Grafana dashboards созданы
- [ ] Alerts для critical events настроены
- [ ] Log aggregation настроен
- [ ] Health checks работают

## Disaster Recovery
- [ ] Backup strategy определена
- [ ] Automated backups настроены
- [ ] Restore procedures документированы
- [ ] DR testing schedule установлен
- [ ] Off-site backup location настроен

## Documentation
- [ ] Architecture diagram создана
- [ ] Operational runbooks написаны
- [ ] Emergency procedures документированы
- [ ] Contact information обновлена
- [ ] Change management process определен

## Compliance
- [ ] Security policy определена
- [ ] Audit requirements выполнены
- [ ] Compliance reports настроены
- [ ] Access reviews scheduled
- [ ] Data retention policy применена

## Testing
- [ ] Load testing выполнено
- [ ] Failover testing выполнено
- [ ] DR restore testing выполнено
- [ ] Security scanning выполнено
- [ ] Performance baselines установлены
CHECKLIST
```

### Финальная проверка

```bash
# Запустите финальную проверку системы
cat > /tmp/final-verification.sh <<'SCRIPT'
#!/bin/bash

echo "╔════════════════════════════════════════════════════════╗"
echo "║     Vault PKI Infrastructure Final Verification        ║"
echo "╚════════════════════════════════════════════════════════╝"
echo ""

CHECKS_PASSED=0
CHECKS_FAILED=0

check() {
  if eval "$2" > /dev/null 2>&1; then
    echo "✓ $1"
    ((CHECKS_PASSED++))
  else
    echo "✗ $1"
    ((CHECKS_FAILED++))
  fi
}

echo "Infrastructure Checks:"
check "Vault pods running" "kubectl -n vault get pods | grep -q Running"
check "Vault unsealed" "kubectl -n vault exec vault-0 -- vault status | grep -q 'Sealed.*false'"
check "cert-manager running" "kubectl -n cert-manager get pods | grep -q Running"
check "Storage provisioned" "kubectl -n vault get pvc | grep -q Bound"
echo ""

echo "PKI Configuration:"
check "Root CA exists" "vault read pki/cert/ca"
check "Intermediate CA exists" "vault read pki_int/cert/ca"
check "PKI roles configured" "vault list pki_int/roles | grep -q local-lab"
check "AppRole configured" "vault read auth/approle/role/cert-manager"
echo ""

echo "Integration Checks:"
check "ClusterIssuer ready" "kubectl get clusterissuer vault-issuer -o json | jq -e '.status.conditions[] | select(.type==\"Ready\" and .status==\"True\")'"
check "Test certificate issued" "kubectl get certificate test-certificate -o json | jq -e '.status.conditions[] | select(.type==\"Ready\" and .status==\"True\")'" || true
echo ""

echo "Monitoring:"
check "ServiceMonitor exists" "kubectl -n vault get servicemonitor vault"
check "PrometheusRule exists" "kubectl -n vault get prometheusrule vault-alerts"
check "Audit logs enabled" "vault audit list | grep -q file"
echo ""

echo "Security:"
check "Network policies applied" "kubectl -n vault get networkpolicy"
check "Pod security policies" "kubectl -n vault get psp vault-psp" || true
check "RBAC configured" "kubectl -n vault get rolebinding"
echo ""

echo "═══════════════════════════════════════════════════════"
echo "Results: $CHECKS_PASSED passed, $CHECKS_FAILED failed"
echo "═══════════════════════════════════════════════════════"

if [ $CHECKS_FAILED -eq 0 ]; then
  echo ""
  echo "🎉 Congratulations! Your Vault PKI infrastructure is ready!"
  echo ""
  echo "Next steps:"
  echo "1. Review the production checklist"
  echo "2. Schedule DR testing"
  echo "3. Set up monitoring alerts"
  echo "4. Document your procedures"
  exit 0
else
  echo ""
  echo "⚠️  Some checks failed. Please review and fix before going to production."
  exit 1
fi
SCRIPT

chmod +x /tmp/final-verification.sh

# Запустите проверку
/tmp/final-verification.sh
```

---

## Поддержка и обратная связь

Если у вас возникли вопросы или проблемы:

1. **Проверьте troubleshooting секцию** этого руководства
2. **Запустите диагностические скрипты** из раздела Troubleshooting
3. **Проверьте логи** Vault и cert-manager
4. **Обратитесь к официальной документации** HashiCorp и cert-manager
5. **Посетите community форумы** для получения помощи

**Важные команды для получения помощи:**

```bash
# Собрать все логи для анализа
kubectl -n vault logs vault-0 > vault-logs.txt
kubectl -n cert-manager logs -l app=cert-manager > certmgr-logs.txt
kubectl get events -A --sort-by='.lastTimestamp' > events.txt

# Создать support bundle
kubectl cluster-info dump --namespaces vault,cert-manager > cluster-dump.txt
```

---

**Поздравляем! Ваша PKI инфраструктура на базе HashiCorp Vault готова к production использованию! 🔐 🚀**

**Версия документа:** 1.0  
**Последнее обновление:** November 2024  
**Автор:** DevOps Team  
**Статус:** Production Ready