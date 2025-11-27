# Руководство по развёртыванию
## Система управления соревнованиями по художественной гимнастике

> **Версия:** 1.0
> **Дата:** 2025-11-26
> **Целевая платформа:** Ubuntu 20.04+ / Docker

---

## Содержание

1. [Требования к системе](#требования-к-системе)
2. [Подготовка окружения](#подготовка-окружения)
3. [Установка зависимостей](#установка-зависимостей)
4. [Настройка базы данных](#настройка-базы-данных)
5. [Развёртывание с Docker](#развёртывание-с-docker)
6. [Развёртывание без Docker](#развёртывание-без-docker)
7. [Настройка Nginx](#настройка-nginx)
8. [SSL/TLS сертификаты](#ssltls-сертификаты)
9. [Локальная сеть (для соревнований)](#локальная-сеть-для-соревнований)
10. [Миграции базы данных](#миграции-базы-данных)
11. [Seed данных](#seed-данных)
12. [Мониторинг и логирование](#мониторинг-и-логирование)
13. [Резервное копирование](#резервное-копирование)
14. [Устранение неполадок](#устранение-неполадок)

---

## Требования к системе

### Минимальные требования (локальная сеть)

**Сервер:**
- CPU: Intel Core i5 / AMD Ryzen 5 (4 ядра)
- RAM: 8 GB
- Storage: 256 GB SSD
- Network: Ethernet 1 Gbps или Wi-Fi 802.11ac (5 GHz)
- OS: Ubuntu 20.04 LTS или macOS 12+

**Рекомендуемое оборудование:**
- Intel NUC 11 / Mac Mini M1
- 16 GB RAM
- 512 GB SSD

### Рекомендуемые требования (облако)

**Production сервер:**
- CPU: 8 vCPU
- RAM: 16 GB
- Storage: 500 GB SSD (RAID 1)
- Network: 1 Gbps
- OS: Ubuntu 22.04 LTS

**Облачные провайдеры:**
- AWS: t3.xlarge или больше
- Azure: Standard_D4s_v3
- Google Cloud: n2-standard-8
- DigitalOcean: $80/mo droplet

---

## Подготовка окружения

### 1. Обновление системы

```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade -y

# macOS
brew update && brew upgrade
```

### 2. Установка необходимых пакетов

```bash
# Ubuntu/Debian
sudo apt install -y \
    curl \
    wget \
    git \
    build-essential \
    ca-certificates \
    gnupg \
    lsb-release

# macOS
xcode-select --install
brew install git curl wget
```

---

## Установка зависимостей

### 1. Node.js (если backend на Node.js)

```bash
# Установка Node.js 18 LTS (через nvm)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc  # или ~/.zshrc для macOS

nvm install 18
nvm use 18
nvm alias default 18

# Проверка версии
node --version  # v18.x.x
npm --version   # 9.x.x
```

### 2. Python (если backend на Python)

```bash
# Ubuntu
sudo apt install -y python3.10 python3.10-venv python3-pip

# macOS
brew install python@3.10

# Создание виртуального окружения
python3 -m venv venv
source venv/bin/activate

# Установка зависимостей
pip install --upgrade pip
pip install -r requirements.txt
```

### 3. PostgreSQL

```bash
# Ubuntu
sudo apt install -y postgresql postgresql-contrib

# Запуск сервиса
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Проверка статуса
sudo systemctl status postgresql

# macOS
brew install postgresql@14
brew services start postgresql@14
```

### 4. Redis

```bash
# Ubuntu
sudo apt install -y redis-server

# Запуск сервиса
sudo systemctl start redis-server
sudo systemctl enable redis-server

# macOS
brew install redis
brew services start redis
```

### 5. Nginx

```bash
# Ubuntu
sudo apt install -y nginx

# Запуск
sudo systemctl start nginx
sudo systemctl enable nginx

# macOS
brew install nginx
brew services start nginx
```

---

## Настройка базы данных

### 1. Создание пользователя и базы данных

```bash
# Войти в PostgreSQL
sudo -u postgres psql

# В PostgreSQL консоли:
CREATE USER rgsystem_user WITH PASSWORD 'SecurePassword123!';
CREATE DATABASE rgsystem_db OWNER rgsystem_user;
GRANT ALL PRIVILEGES ON DATABASE rgsystem_db TO rgsystem_user;

# Выйти
\q
```

### 2. Настройка конфигурации PostgreSQL

```bash
# Редактировать postgresql.conf
sudo nano /etc/postgresql/14/main/postgresql.conf

# Изменить параметры:
listen_addresses = 'localhost'
max_connections = 100
shared_buffers = 256MB
effective_cache_size = 1GB
work_mem = 16MB
```

### 3. Настройка pg_hba.conf

```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf

# Добавить:
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   rgsystem_db     rgsystem_user                           md5
host    rgsystem_db     rgsystem_user   127.0.0.1/32            md5
```

### 4. Перезапуск PostgreSQL

```bash
sudo systemctl restart postgresql
```

### 5. Тестирование подключения

```bash
psql -U rgsystem_user -d rgsystem_db -h localhost

# Должен запросить пароль, затем войти в базу
```

---

## Развёртывание с Docker

### 1. Установка Docker

```bash
# Ubuntu
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Добавить текущего пользователя в группу docker
sudo usermod -aG docker $USER
newgrp docker

# Установка Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Проверка версий
docker --version
docker-compose --version
```

### 2. Создание docker-compose.yml

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:14-alpine
    container_name: rgsystem_postgres
    environment:
      POSTGRES_DB: rgsystem_db
      POSTGRES_USER: rgsystem_user
      POSTGRES_PASSWORD: SecurePassword123!
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - rgsystem_network
    restart: unless-stopped

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: rgsystem_redis
    command: redis-server --requirepass RedisPassword123!
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - rgsystem_network
    restart: unless-stopped

  # API Server (Node.js example)
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: rgsystem_api
    environment:
      NODE_ENV: production
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: rgsystem_db
      DB_USER: rgsystem_user
      DB_PASSWORD: SecurePassword123!
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: RedisPassword123!
      JWT_SECRET: YourSuperSecretJWTKey123!
      PORT: 3000
    ports:
      - "3000:3000"
    depends_on:
      - postgres
      - redis
    networks:
      - rgsystem_network
    restart: unless-stopped
    volumes:
      - ./uploads:/app/uploads
      - ./logs:/app/logs

  # WebSocket Server
  websocket:
    build:
      context: ./websocket
      dockerfile: Dockerfile
    container_name: rgsystem_websocket
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: RedisPassword123!
      PORT: 3001
    ports:
      - "3001:3001"
    depends_on:
      - redis
    networks:
      - rgsystem_network
    restart: unless-stopped

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: rgsystem_nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - ./frontend/dist:/usr/share/nginx/html:ro
    depends_on:
      - api
      - websocket
    networks:
      - rgsystem_network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  rgsystem_network:
    driver: bridge
```

### 3. Создание .env файла

```bash
# Создать файл .env
cat > .env << 'EOF'
# Database
DB_HOST=postgres
DB_PORT=5432
DB_NAME=rgsystem_db
DB_USER=rgsystem_user
DB_PASSWORD=SecurePassword123!

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=RedisPassword123!

# Application
NODE_ENV=production
PORT=3000
JWT_SECRET=YourSuperSecretJWTKey123!
JWT_EXPIRES_IN=1h

# Frontend
REACT_APP_API_URL=https://api.rgsystem.example
REACT_APP_WS_URL=wss://api.rgsystem.example/ws
EOF
```

### 4. Dockerfile для API (пример Node.js)

```dockerfile
# backend/Dockerfile
FROM node:18-alpine

WORKDIR /app

# Копирование package files
COPY package*.json ./

# Установка зависимостей
RUN npm ci --only=production

# Копирование исходного кода
COPY . .

# Сборка TypeScript (если используется)
RUN npm run build

# Экспозиция порта
EXPOSE 3000

# Запуск приложения
CMD ["node", "dist/index.js"]
```

### 5. Запуск всех сервисов

```bash
# Сборка образов
docker-compose build

# Запуск в фоновом режиме
docker-compose up -d

# Просмотр логов
docker-compose logs -f

# Проверка статуса
docker-compose ps
```

### 6. Остановка и удаление

```bash
# Остановка
docker-compose down

# Остановка с удалением volumes (ВНИМАНИЕ: удалит данные!)
docker-compose down -v
```

---

## Развёртывание без Docker

### 1. Клонирование репозитория

```bash
git clone https://github.com/your-org/rgsystem.git
cd rgsystem
```

### 2. Backend (Node.js пример)

```bash
cd backend

# Установка зависимостей
npm install

# Создание .env файла
cp .env.example .env
nano .env  # Отредактировать настройки

# Запуск миграций
npm run migrate

# Seed данных (опционально)
npm run seed

# Сборка (если TypeScript)
npm run build

# Запуск в production режиме
npm run start

# Или с PM2 (рекомендуется)
npm install -g pm2
pm2 start dist/index.js --name rgsystem-api
pm2 save
pm2 startup
```

### 3. Frontend

```bash
cd frontend

# Установка зависимостей
npm install

# Сборка для production
npm run build

# Копирование в директорию Nginx
sudo cp -r dist/* /var/www/html/rgsystem/
```

### 4. WebSocket Server

```bash
cd websocket

npm install
npm run build

# Запуск с PM2
pm2 start dist/websocket.js --name rgsystem-ws
```

---

## Настройка Nginx

### 1. Создание конфигурации

```bash
sudo nano /etc/nginx/sites-available/rgsystem
```

### 2. Конфигурация Nginx

```nginx
# Upstream для API
upstream api_backend {
    server 127.0.0.1:3000;
    # Если несколько серверов:
    # server 127.0.0.1:3001;
    # server 127.0.0.1:3002;
}

# Upstream для WebSocket
upstream websocket_backend {
    server 127.0.0.1:3001;
}

# HTTP -> HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name rgsystem.example.com;

    return 301 https://$server_name$request_uri;
}

# HTTPS Server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name rgsystem.example.com;

    # SSL Configuration
    ssl_certificate /etc/nginx/ssl/rgsystem.crt;
    ssl_certificate_key /etc/nginx/ssl/rgsystem.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Frontend (React/Vue/Angular)
    root /var/www/html/rgsystem;
    index index.html;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;

    # API Proxy
    location /api/ {
        proxy_pass http://api_backend/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }

    # WebSocket Proxy
    location /ws {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # WebSocket specific
        proxy_read_timeout 86400;
    }

    # Frontend SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Static files caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### 3. Активация конфигурации

```bash
# Создать символическую ссылку
sudo ln -s /etc/nginx/sites-available/rgsystem /etc/nginx/sites-enabled/

# Проверить конфигурацию
sudo nginx -t

# Перезапустить Nginx
sudo systemctl restart nginx
```

---

## SSL/TLS сертификаты

### 1. Установка Certbot (Let's Encrypt)

```bash
# Ubuntu
sudo apt install -y certbot python3-certbot-nginx

# Получение сертификата
sudo certbot --nginx -d rgsystem.example.com

# Автоматическое обновление
sudo systemctl status certbot.timer
```

### 2. Самоподписанный сертификат (для тестирования)

```bash
# Создание директории
sudo mkdir -p /etc/nginx/ssl

# Генерация сертификата
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/rgsystem.key \
  -out /etc/nginx/ssl/rgsystem.crt

# Ввести данные:
# Country: RU
# State: Moscow
# Locality: Moscow
# Organization: RG System
# Common Name: rgsystem.local
```

---

## Локальная сеть (для соревнований)

### 1. Настройка Wi-Fi роутера

**Рекомендуемые параметры:**
- SSID: `RGSystem_Competition`
- Пароль: Длинный и сложный (минимум 16 символов)
- Частота: 5 GHz (802.11ac)
- Канал: Автоматически или фиксированный (36, 40, 44, 48)
- Безопасность: WPA3 или WPA2-PSK

### 2. Статический IP для сервера

```bash
# Ubuntu (netplan)
sudo nano /etc/netplan/01-netcfg.yaml

# Добавить:
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.1.100/24
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]

# Применить
sudo netplan apply
```

### 3. Настройка mDNS (для локального доступа)

```bash
# Установка Avahi (Ubuntu)
sudo apt install -y avahi-daemon

# Теперь доступ по: http://rgsystem.local
```

### 4. Локальный DNS (опционально)

```bash
# Установка dnsmasq
sudo apt install -y dnsmasq

# Конфигурация
sudo nano /etc/dnsmasq.conf

# Добавить:
address=/rgsystem.local/192.168.1.100
```

---

## Миграции базы данных

### 1. Применение миграций (Sequelize пример)

```bash
# Создание новой миграции
npx sequelize-cli migration:generate --name create-competitions-table

# Применение всех миграций
npx sequelize-cli db:migrate

# Откат последней миграции
npx sequelize-cli db:migrate:undo

# Откат всех миграций
npx sequelize-cli db:migrate:undo:all
```

### 2. Применение миграций (TypeORM пример)

```bash
# Создание миграции
npm run typeorm migration:generate -- -n CreateCompetitionsTable

# Применение
npm run typeorm migration:run

# Откат
npm run typeorm migration:revert
```

### 3. Применение миграций (Prisma пример)

```bash
# Создание миграции
npx prisma migrate dev --name init

# Применение в production
npx prisma migrate deploy
```

---

## Seed данных

### 1. Создание seed скрипта

```javascript
// seeds/01-users.js
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.bulkInsert('users', [
      {
        email: 'admin@rgsystem.local',
        password_hash: '$2b$12$...', // bcrypt hash
        first_name: 'Admin',
        last_name: 'User',
        role: 'admin',
        is_active: true,
        created_at: new Date(),
        updated_at: new Date()
      }
    ]);
  },

  down: async (queryInterface, Sequelize) => {
    await queryInterface.bulkDelete('users', null, {});
  }
};
```

### 2. Запуск seed

```bash
npx sequelize-cli db:seed:all
```

---

## Мониторинг и логирование

### 1. Установка Prometheus + Grafana

```bash
# Docker Compose для мониторинга
docker-compose -f docker-compose.monitoring.yml up -d
```

### 2. PM2 мониторинг

```bash
# Установка PM2 Plus (бесплатно для open source)
pm2 plus

# Просмотр логов
pm2 logs

# Мониторинг
pm2 monit
```

---

## Резервное копирование

### 1. Скрипт автоматического бэкапа

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR="/backup/rgsystem"
DATE=$(date +"%Y%m%d_%H%M%S")

# Создание директории
mkdir -p $BACKUP_DIR

# Бэкап PostgreSQL
pg_dump -U rgsystem_user -h localhost rgsystem_db | gzip > "$BACKUP_DIR/db_$DATE.sql.gz"

# Бэкап файлов
tar -czf "$BACKUP_DIR/files_$DATE.tar.gz" /var/www/html/rgsystem/uploads

# Удаление старых бэкапов (старше 30 дней)
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete

echo "Backup completed: $DATE"
```

### 2. Настройка cron

```bash
# Редактировать crontab
crontab -e

# Добавить (бэкап каждые 6 часов)
0 */6 * * * /opt/scripts/backup.sh >> /var/log/rgsystem-backup.log 2>&1
```

---

## Устранение неполадок

### Проблема 1: База данных недоступна

```bash
# Проверка статуса
sudo systemctl status postgresql

# Проверка логов
sudo tail -f /var/log/postgresql/postgresql-14-main.log

# Перезапуск
sudo systemctl restart postgresql
```

### Проблема 2: API не отвечает

```bash
# Проверка процесса
pm2 list

# Просмотр логов
pm2 logs rgsystem-api --lines 100

# Перезапуск
pm2 restart rgsystem-api
```

### Проблема 3: Nginx 502 Bad Gateway

```bash
# Проверка upstream серверов
curl http://localhost:3000/api/health

# Проверка логов Nginx
sudo tail -f /var/log/nginx/error.log

# Перезапуск Nginx
sudo systemctl restart nginx
```

### Проблема 4: WebSocket не подключается

```bash
# Проверка порта
netstat -tulnp | grep 3001

# Тест WebSocket
wscat -c ws://localhost:3001

# Проверка Nginx конфигурации
sudo nginx -t
```

---

## Чеклист развёртывания

### Перед развёртыванием:
- [ ] Сервер соответствует минимальным требованиям
- [ ] Доменное имя настроено (если облако)
- [ ] SSL сертификат получен
- [ ] .env файл создан и заполнен
- [ ] Бэкап стратегия определена

### Развёртывание:
- [ ] Зависимости установлены
- [ ] База данных создана и настроена
- [ ] Миграции применены
- [ ] Seed данные загружены (опционально)
- [ ] Приложение запущено
- [ ] Nginx настроен и запущен
- [ ] SSL работает

### После развёртывания:
- [ ] Проверка всех API эндпоинтов
- [ ] Проверка WebSocket подключения
- [ ] Вход в систему работает
- [ ] Мониторинг настроен
- [ ] Бэкап автоматизирован
- [ ] Логи ротируются

---

**Конец руководства**

> **Поддержка:** Если возникли проблемы, создайте issue в репозитории
> **Обновления:** Проверяйте обновления документации регулярно
