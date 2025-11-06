# Настройка Smallstep Certificate Authority для DevOps инфраструктуры

## 📋 Содержание

- [О Smallstep CA](#о-smallstep-ca)
- [Преимущества использования](#преимущества-использования)
- [Архитектура интеграции](#архитектура-интеграции)
- [Установка Smallstep CA](#установка-smallstep-ca)
- [Настройка для Harbor](#настройка-для-harbor)
- [Настройка для Monitoring Server](#настройка-для-monitoring-server)
- [Интеграция с K3s](#интеграция-с-k3s)
- [Автоматическое обновление сертификатов](#автоматическое-обновление-сертификатов)
- [Мониторинг и управление](#мониторинг-и-управление)
- [Troubleshooting](#troubleshooting)

---

## 🔐 О Smallstep CA

**Smallstep Certificate Authority** - это современный, автоматизированный центр сертификации для управления X.509 и SSH сертификатами в инфраструктуре.

### Основные возможности

✅ Автоматическая выдача и обновление сертификатов  
✅ Поддержка ACME протокола (как Let's Encrypt)  
✅ Встроенная ротация сертификатов  
✅ API для автоматизации  
✅ Интеграция с CI/CD  
✅ Централизованное управление PKI  
✅ Поддержка короткоживущих сертификатов (short-lived certificates)  

---

## 🎯 Преимущества использования

### Для нашей инфраструктуры

**Вместо самоподписанных сертификатов:**
- ❌ Ручное создание и обновление
- ❌ Проблемы с истечением срока действия
- ❌ Сложность распространения CA сертификата
- ❌ Отсутствие централизованного управления

**Со Smallstep CA:**
- ✅ Автоматическое обновление сертификатов
- ✅ Единый корневой сертификат для всей инфраструктуры
- ✅ Мониторинг истечения сертификатов
- ✅ Простая интеграция с Docker и Kubernetes
- ✅ Возможность отзыва сертификатов
- ✅ Audit log всех операций

---

## 🏗️ Архитектура интеграции

```
┌─────────────────────────────────────────────────────────┐
│              Smallstep CA Server                        │
│              (192.168.100.50)                           │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Root CA Certificate                             │  │
│  │  - Срок действия: 10 лет                        │  │
│  │  - Используется для подписи всех сертификатов   │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │  Intermediate CA                                 │  │
│  │  - Выдает сертификаты для сервисов              │  │
│  │  - Автоматическое обновление                     │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          ↓
         ┌────────────────┼────────────────┐
         ↓                ↓                ↓
    ┌─────────┐      ┌──────────┐    ┌──────────┐
    │ Harbor  │      │Monitoring│    │   K3s    │
    │  :32    │      │   :40    │    │ Cluster  │
    └─────────┘      └──────────┘    └──────────┘
         ↓                ↓                ↓
    Auto-renew      Auto-renew       Auto-renew
    каждые 30д      каждые 30д       каждые 90д
```

---

## 📦 Установка Smallstep CA

### Этап 1: Создание VM для CA

**Характеристики:**
- IP: 192.168.100.50
- CPU: 2 cores
- RAM: 2GB
- Disk: 20GB
- OS: Ubuntu 22.04

**Создание через Terraform или вручную в Proxmox**

### Этап 2: Установка Smallstep

```bash
# Подключение к CA серверу
ssh ubuntu@192.168.100.50

# Обновление системы
sudo apt update && sudo apt upgrade -y

# Установка зависимостей
sudo apt install -y curl wget jq

# Скачивание и установка step-ca
wget https://dl.smallstep.com/gh-release/cli/docs-ca-install/v0.25.0/step-cli_0.25.0_amd64.deb
sudo dpkg -i step-cli_0.25.0_amd64.deb

# Установка CA сервера
wget https://dl.smallstep.com/gh-release/certificates/docs-ca-install/v0.25.0/step-ca_0.25.0_amd64.deb
sudo dpkg -i step-ca_0.25.0_amd64.deb

# Проверка установки
step version
step-ca version
```

### Этап 3: Инициализация CA

```bash
# Создание директории для CA
sudo mkdir -p /etc/step-ca
sudo chown -R ubuntu:ubuntu /etc/step-ca
cd /etc/step-ca

# Инициализация CA
step ca init \
  --name="DevOps Internal CA" \
  --dns="ca.local.lab" \
  --address=":443" \
  --provisioner="admin" \
  --password-file=/dev/stdin <<< "YourSecurePassword123!"

# Сохраните пароль в безопасном месте!
```

**Результат инициализации:**
```
✔ Root certificate: /etc/step-ca/certs/root_ca.crt
✔ Root private key: /etc/step-ca/secrets/root_ca_key
✔ Root fingerprint: abc123def456...
✔ Intermediate certificate: /etc/step-ca/certs/intermediate_ca.crt
✔ Intermediate private key: /etc/step-ca/secrets/intermediate_ca_key
✔ Database folder: /etc/step-ca/db
✔ Default configuration: /etc/step-ca/config/ca.json
✔ Certificate Authority configuration has been saved in /etc/step-ca/config/ca.json
```

### Этап 4: Настройка CA конфигурации

```bash
# Редактирование конфигурации
sudo tee /etc/step-ca/config/ca.json > /dev/null <<'EOF'
{
  "root": "/etc/step-ca/certs/root_ca.crt",
  "federatedRoots": null,
  "crt": "/etc/step-ca/certs/intermediate_ca.crt",
  "key": "/etc/step-ca/secrets/intermediate_ca_key",
  "address": ":443",
  "dnsNames": [
    "ca.local.lab",
    "192.168.100.50"
  ],
  "logger": {
    "format": "text"
  },
  "db": {
    "type": "badgerv2",
    "dataSource": "/etc/step-ca/db"
  },
  "authority": {
    "provisioners": [
      {
        "type": "JWK",
        "name": "admin",
        "key": {
          "use": "sig",
          "kty": "EC",
          "kid": "...",
          "crv": "P-256",
          "alg": "ES256",
          "x": "...",
          "y": "..."
        },
        "encryptedKey": "..."
      },
      {
        "type": "ACME",
        "name": "acme",
        "claims": {
          "minTLSCertDuration": "5m",
          "maxTLSCertDuration": "2160h",
          "defaultTLSCertDuration": "2160h"
        }
      }
    ]
  },
  "tls": {
    "cipherSuites": [
      "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305",
      "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
    ],
    "minVersion": 1.2,
    "maxVersion": 1.3,
    "renegotiation": false
  }
}
EOF
```

### Этап 5: Создание systemd сервиса

```bash
sudo tee /etc/systemd/system/step-ca.service > /dev/null <<'EOF'
[Unit]
Description=Smallstep Certificate Authority
After=network.target
Documentation=https://smallstep.com/docs/step-ca

[Service]
Type=simple
User=ubuntu
Group=ubuntu
Environment="STEPPATH=/etc/step-ca"
WorkingDirectory=/etc/step-ca
ExecStart=/usr/bin/step-ca /etc/step-ca/config/ca.json --password-file=/etc/step-ca/secrets/password
Restart=always
RestartSec=10
StandardOutput=append:/var/log/step-ca/output.log
StandardError=append:/var/log/step-ca/error.log

[Install]
WantedBy=multi-user.target
EOF

# Создание директории для логов
sudo mkdir -p /var/log/step-ca
sudo chown ubuntu:ubuntu /var/log/step-ca

# Сохранение пароля CA
echo "YourSecurePassword123!" | sudo tee /etc/step-ca/secrets/password > /dev/null
sudo chmod 600 /etc/step-ca/secrets/password

# Запуск CA сервера
sudo systemctl daemon-reload
sudo systemctl enable step-ca
sudo systemctl start step-ca

# Проверка статуса
sudo systemctl status step-ca
```

### Этап 6: Добавление DNS записи

На DNS сервере (192.168.100.53):

```bash
ssh ubuntu@192.168.100.53

# Добавление в зону
sudo tee -a /etc/bind/db.local.lab > /dev/null <<'EOF'
ca              IN      A       192.168.100.50
EOF

# Обновление Serial и перезагрузка
sudo rndc reload
```

### Этап 7: Проверка работы CA

```bash
# На CA сервере
step ca health

# Получение root сертификата
step ca root root_ca.crt

# Просмотр информации о CA
step certificate inspect root_ca.crt
```

---

## 🐳 Настройка для Harbor

### Этап 1: Установка step-cli на Harbor сервере

```bash
ssh ubuntu@192.168.100.32

# Установка step-cli
wget https://dl.smallstep.com/gh-release/cli/docs-ca-install/v0.25.0/step-cli_0.25.0_amd64.deb
sudo dpkg -i step-cli_0.25.0_amd64.deb

# Bootstrap к CA серверу
step ca bootstrap \
  --ca-url https://ca.local.lab \
  --fingerprint $(ssh ubuntu@192.168.100.50 "step certificate fingerprint /etc/step-ca/certs/root_ca.crt")

# Установка root сертификата
sudo step ca root /usr/local/share/ca-certificates/step-ca-root.crt
sudo update-ca-certificates
```

### Этап 2: Получение сертификата для Harbor

```bash
# Создание директории для сертификатов
sudo mkdir -p /etc/harbor/certs
cd /etc/harbor/certs

# Запрос сертификата от CA
step ca certificate harbor.local.lab \
  harbor.crt harbor.key \
  --san harbor.local.lab \
  --san 192.168.100.32 \
  --san localhost \
  --not-after 2160h

# Установка правильных прав
sudo chmod 644 harbor.crt
sudo chmod 600 harbor.key
sudo chown -R ubuntu:ubuntu /etc/harbor/certs
```

### Этап 3: Обновление конфигурации Harbor

```bash
cd ~/harbor

# Обновление harbor.yml
sed -i 's|certificate: .*|certificate: /etc/harbor/certs/harbor.crt|' harbor.yml
sed -i 's|private_key: .*|private_key: /etc/harbor/certs/harbor.key|' harbor.yml

# Пересоздание конфигурации
sudo ./prepare

# Перезапуск Harbor
sudo docker-compose down
sudo docker-compose up -d
```

### Этап 4: Настройка автоматического обновления

```bash
# Создание скрипта обновления
sudo tee /usr/local/bin/renew-harbor-cert.sh > /dev/null <<'EOF'
#!/bin/bash

cd /etc/harbor/certs

# Обновление сертификата
step ca renew harbor.crt harbor.key --force

# Перезапуск Harbor
cd ~/harbor
docker-compose restart
EOF

sudo chmod +x /usr/local/bin/renew-harbor-cert.sh

# Настройка cron для автоматического обновления
crontab -e
# Добавить: обновление каждые 30 дней
0 2 1 * * /usr/local/bin/renew-harbor-cert.sh >> /var/log/harbor-cert-renew.log 2>&1
```

### Этап 5: Обновление доверенных сертификатов на клиентах

**На Jenkins сервере и K3s нодах:**

```bash
# Копирование root сертификата CA
sudo scp ubuntu@ca.local.lab:/etc/step-ca/certs/root_ca.crt \
  /usr/local/share/ca-certificates/step-ca-root.crt

sudo update-ca-certificates

# Для Docker
sudo mkdir -p /etc/docker/certs.d/harbor.local.lab
sudo cp /usr/local/share/ca-certificates/step-ca-root.crt \
  /etc/docker/certs.d/harbor.local.lab/ca.crt

# Перезапуск Docker
sudo systemctl restart docker

# Проверка
docker pull harbor.local.lab/library/test:latest
```

---

## 📊 Настройка для Monitoring Server

### Этап 1: Получение сертификатов

```bash
ssh ubuntu@192.168.100.40

# Установка step-cli
wget https://dl.smallstep.com/gh-release/cli/docs-ca-install/v0.25.0/step-cli_0.25.0_amd64.deb
sudo dpkg -i step-cli_0.25.0_amd64.deb

# Bootstrap к CA
step ca bootstrap \
  --ca-url https://ca.local.lab \
  --fingerprint $(ssh ubuntu@192.168.100.50 "step certificate fingerprint /etc/step-ca/certs/root_ca.crt")

# Создание директории
sudo mkdir -p /etc/monitoring/certs

# Запрос multi-domain сертификата
step ca certificate monitoring.local.lab \
  /etc/monitoring/certs/monitoring.crt \
  /etc/monitoring/certs/monitoring.key \
  --san monitoring.local.lab \
  --san grafana.local.lab \
  --san prometheus.local.lab \
  --san 192.168.100.40 \
  --not-after 2160h

# Права
sudo chmod 644 /etc/monitoring/certs/monitoring.crt
sudo chmod 600 /etc/monitoring/certs/monitoring.key
```

### Этап 2: Обновление nginx конфигурации

```bash
sudo tee /etc/nginx/sites-available/monitoring.conf > /dev/null <<'EOF'
# Grafana
server {
    listen 443 ssl http2;
    server_name grafana.local.lab monitoring.local.lab;

    ssl_certificate /etc/monitoring/certs/monitoring.crt;
    ssl_certificate_key /etc/monitoring/certs/monitoring.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# Prometheus
server {
    listen 443 ssl http2;
    server_name prometheus.local.lab;

    ssl_certificate /etc/monitoring/certs/monitoring.crt;
    ssl_certificate_key /etc/monitoring/certs/monitoring.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://127.0.0.1:9090;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}

# HTTP to HTTPS redirect
server {
    listen 80;
    server_name grafana.local.lab prometheus.local.lab monitoring.local.lab;
    return 301 https://$host$request_uri;
}
EOF

# Перезапуск nginx
sudo nginx -t
sudo systemctl reload nginx
```

### Этап 3: Автоматическое обновление

```bash
sudo tee /usr/local/bin/renew-monitoring-cert.sh > /dev/null <<'EOF'
#!/bin/bash

step ca renew /etc/monitoring/certs/monitoring.crt \
  /etc/monitoring/certs/monitoring.key --force

nginx -s reload
EOF

sudo chmod +x /usr/local/bin/renew-monitoring-cert.sh

# Cron задача
crontab -e
# Добавить:
0 2 1 * * /usr/local/bin/renew-monitoring-cert.sh >> /var/log/monitoring-cert-renew.log 2>&1
```

---

## ☸️ Интеграция с K3s

### Этап 1: Установка step на K3s нодах

**На всех нодах (master + workers):**

```bash
# Установка step-cli
wget https://dl.smallstep.com/gh-release/cli/docs-ca-install/v0.25.0/step-cli_0.25.0_amd64.deb
sudo dpkg -i step-cli_0.25.0_amd64.deb

# Bootstrap
step ca bootstrap \
  --ca-url https://ca.local.lab \
  --fingerprint $(ssh ubuntu@192.168.100.50 "step certificate fingerprint /etc/step-ca/certs/root_ca.crt")

# Установка root CA
sudo step ca root /usr/local/share/ca-certificates/step-ca-root.crt
sudo update-ca-certificates
```

### Этап 2: Создание Kubernetes Secret с CA сертификатом

```bash
# На jumphost или k3s-master
kubectl create secret generic step-ca-root \
  --from-file=ca.crt=/usr/local/share/ca-certificates/step-ca-root.crt \
  -n kube-system

# Для Harbor registry
kubectl create secret generic step-ca-root \
  --from-file=ca.crt=/usr/local/share/ca-certificates/step-ca-root.crt \
  -n production
```

### Этап 3: Настройка автоматического обновления в K3s

```bash
# Создание ConfigMap для step configuration
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: step-ca-config
  namespace: kube-system
data:
  ca-url: "https://ca.local.lab"
  root: |
$(cat /usr/local/share/ca-certificates/step-ca-root.crt | sed 's/^/    /')
EOF
```

---

## 🔄 Автоматическое обновление сертификатов

### Централизованный скрипт мониторинга

На CA сервере создайте скрипт для мониторинга всех сертификатов:

```bash
sudo tee /usr/local/bin/monitor-certs.sh > /dev/null <<'EOF'
#!/bin/bash

ALERT_DAYS=30
LOG_FILE="/var/log/step-ca/cert-monitor.log"

echo "=== Certificate Monitor - $(date) ===" >> $LOG_FILE

# Проверка всех выданных сертификатов
step ca admin list certificates | while read cert; do
  EXPIRY=$(step certificate inspect $cert --format json | jq -r '.validity.end')
  DAYS_LEFT=$(( ($(date -d "$EXPIRY" +%s) - $(date +%s)) / 86400 ))
  
  if [ $DAYS_LEFT -lt $ALERT_DAYS ]; then
    echo "WARNING: Certificate $cert expires in $DAYS_LEFT days" >> $LOG_FILE
    # Отправка уведомления (опционально)
    # mail -s "Certificate Expiry Warning" admin@example.com < /tmp/alert.txt
  fi
done

echo "=== Monitor Complete ===" >> $LOG_FILE
EOF

sudo chmod +x /usr/local/bin/monitor-certs.sh

# Добавление в cron
sudo crontab -e
# Добавить: проверка каждый день в 9:00
0 9 * * * /usr/local/bin/monitor-certs.sh
```

### Автоматизация через step-ca renewal

```bash
# Настройка автоматического продления на каждом сервере
cat > /etc/systemd/system/step-renew.service <<'EOF'
[Unit]
Description=Step Certificate Renewal
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/bin/step ca renew --force /path/to/cert.crt /path/to/cert.key
ExecStartPost=/bin/systemctl reload service-name

[Install]
WantedBy=multi-user.target
EOF

# Timer для периодического запуска
cat > /etc/systemd/system/step-renew.timer <<'EOF'
[Unit]
Description=Run step certificate renewal monthly

[Timer]
OnCalendar=monthly
Persistent=true

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now step-renew.timer
```

---

## 📊 Мониторинг и управление

### Интеграция с Prometheus

```yaml
# Добавить в prometheus.yml
scrape_configs:
  - job_name: 'step-ca'
    static_configs:
      - targets: ['ca.local.lab:443']
    metrics_path: '/metrics'
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/step-ca-root.crt
```

### Grafana Dashboard для сертификатов

```bash
# Создание custom dashboard для мониторинга сертификатов
# Import Dashboard ID: 12345 (создайте свой или используйте community dashboard)
```

### Проверка статуса всех сертификатов

```bash
# На CA сервере
step ca admin list certificates --all

# Детальная информация
step certificate inspect /path/to/cert.crt

# Проверка цепочки
step certificate verify /path/to/cert.crt --roots /etc/step-ca/certs/root_ca.crt
```

---

## 🔧 Troubleshooting

### Проблема: CA сервер недоступен

```bash
# Проверка статуса
sudo systemctl status step-ca

# Логи
sudo journalctl -u step-ca -f

# Проверка порта
sudo netstat -tulpn | grep 443

# Проверка сертификата CA
step certificate inspect /etc/step-ca/certs/intermediate_ca.crt
```

### Проблема: Сертификат не обновляется

```bash
# Проверка подключения к CA
step ca health

# Ручное обновление
step ca renew cert.crt cert.key --force

# Проверка прав на файлы
ls -la /path/to/certs/

# Проверка cron задач
crontab -l
sudo tail -f /var/log/syslog | grep CRON
```

### Проблема: Docker не доверяет сертификату Harbor

```bash
# Проверка сертификатов Docker
ls -la /etc/docker/certs.d/harbor.local.lab/

# Переустановка root CA
sudo cp /usr/local/share/ca-certificates/step-ca-root.crt \
  /etc/docker/certs.d/harbor.local.lab/ca.crt

sudo systemctl restart docker

# Проверка
docker info | grep -A 10 "Registry"
```

### Проблема: Nginx возвращает SSL ошибку

```bash
# Проверка конфигурации
sudo nginx -t

# Проверка сертификата
openssl s_client -connect monitoring.local.lab:443 -showcerts

# Проверка прав
ls -la /etc/monitoring/certs/

# Просмотр логов
sudo tail -f /var/log/nginx/error.log
```

---

## 📚 Полезные команды

### Управление сертификатами

```bash
# Получение нового сертификата
step ca certificate example.local.lab cert.crt cert.key

# Обновление существующего
step ca renew cert.crt cert.key

# Принудительное обновление
step ca renew cert.crt cert.key --force

# Отзыв сертификата
step ca revoke --cert cert.crt --key cert.key

# Проверка сертификата
step certificate inspect cert.crt

# Проверка цепочки доверия
step certificate verify cert.crt --roots root_ca.crt
```

### Управление CA

```bash
# Статус CA
step ca health

# Информация о provisioner
step ca provisioner list

# Добавление нового provisioner
step ca provisioner add acme --type ACME

# Просмотр логов
sudo journalctl -u step-ca -n 100 -f
```

### Backup и восстановление

```bash
# Backup CA
sudo tar czf step-ca-backup-$(date +%Y%m%d).tar.gz \
  /etc/step-ca/certs \
  /etc/step-ca/secrets \
  /etc/step-ca/config \
  /etc/step-ca/db

# Восстановление
sudo tar xzf step-ca-backup-*.tar.gz -C /
sudo systemctl restart step-ca
```

---

## ✅ Чеклист после установки

- [ ] CA сервер запущен и доступен
- [ ] Root сертификат установлен на всех серверах
- [ ] Harbor использует сертификаты от CA
- [ ] Monitoring сервер использует сертификаты от CA
- [ ] K3s ноды доверяют CA
- [ ] Docker на всех нодах доверяет Harbor
- [ ] Настроено автоматическое обновление сертификатов
- [ ] Настроен мониторинг истечения сертификатов
- [ ] Создан backup CA
- [ ] Документированы все пароли и ключи

---

## 🎯 Результат

После завершения установки у вас будет:

✅ Полностью автоматизированный PKI  
✅ Централизованное управление сертификатами  
✅ Автоматическое обновление перед истечением  
✅ Единый root CA для всей инфраструктуры  
✅ Мониторинг состояния сертификатов  
✅ Безопасная HTTPS связь между всеми сервисами  

**🎉 Ваша DevOps инфраструктура теперь использует enterprise-grade PKI!**

---

*Последнее обновление: 2025*  
*Версия: 1.0.0*  
*Совместимость: Ubuntu 22.04, Smallstep CA 0.25.0+*