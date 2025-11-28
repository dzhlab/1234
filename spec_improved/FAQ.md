# Часто задаваемые вопросы (FAQ)
# Frequently Asked Questions

## Содержание / Table of Contents

1. [Общие вопросы](#общие-вопросы--general-questions)
2. [Установка и развертывание](#установка-и-развертывание--installation-and-deployment)
3. [Конфигурация](#конфигурация--configuration)
4. [Судейство и оценки](#судейство-и-оценки--judging-and-scoring)
5. [Технические вопросы](#технические-вопросы--technical-questions)
6. [Безопасность](#безопасность--security)
7. [Производительность](#производительность--performance)
8. [Устранение неполадок](#устранение-неполадок--troubleshooting)
9. [Интеграция и API](#интеграция-и-api--integration-and-api)
10. [Лицензирование и поддержка](#лицензирование-и-поддержка--licensing-and-support)

---

## Общие вопросы / General Questions

### В1: Что такое RG System?

**RU:** RG System — это комплексная информационная система для автоматизации организации и проведения соревнований по художественной гимнастике. Система поддерживает правила FIG Code of Points 2025-2028 и предоставляет инструменты для организаторов, судей, секретарей и зрителей.

**EN:** RG System is a comprehensive information system for automating the organization and conduct of rhythmic gymnastics competitions. The system supports FIG Code of Points 2025-2028 rules and provides tools for organizers, judges, secretaries, and viewers.

### В2: Кто может использовать систему?

Система предназначена для:
- **Организаторов** — создание и управление соревнованиями
- **Главных судей** — управление судейской коллегией и валидация оценок
- **Судей** — выставление оценок D, E, A, ND
- **Секретарей** — управление расписанием и протоколами
- **Спортсменов** — регистрация и просмотр результатов
- **Зрителей** — просмотр результатов в реальном времени

### В3: Соответствует ли система международным стандартам FIG?

Да, система полностью соответствует правилам **FIG Code of Points 2025-2028**, включая:
- Расчет D-score (DB + DA + DS + DD)
- Расчет E-score с исключением крайних значений
- Расчет A-score (артистизм)
- Нейтральные сбавки (ND)
- Алгоритм разрешения равенства (тайбрейк)

### В4: Можно ли использовать систему без интернета?

Да, система поддерживает **офлайн-режим**:
- Локальное хранилище данных в браузере
- Автоматическая синхронизация при восстановлении соединения
- Локальное развертывание на площадке соревнований
- Работа в локальной сети без доступа в интернет

### В5: На каких языках доступна система?

В настоящее время система поддерживает:
- 🇷🇺 **Русский** (основной язык)
- 🇬🇧 **Английский** (в разработке)

Планируется добавление дополнительных языков в будущих версиях.

---

## Установка и развертывание / Installation and Deployment

### В6: Какие системные требования для развертывания?

**Минимальные требования:**
- CPU: 4 ядра (2.0 GHz+)
- RAM: 8 GB
- HDD: 256 GB SSD
- ОС: Ubuntu 20.04 LTS / Debian 11 / CentOS 8 / Windows Server 2019

**Рекомендуемые требования:**
- CPU: 8 ядер (3.0 GHz+)
- RAM: 16 GB
- HDD: 512 GB NVMe SSD
- ОС: Ubuntu 22.04 LTS

### В7: Как установить систему с помощью Docker?

1. Клонируйте репозиторий:
   ```bash
   git clone https://github.com/your-org/rgsystem.git
   cd rgsystem
   ```

2. Скопируйте и настройте переменные окружения:
   ```bash
   cp spec_improved/.env.example .env
   nano .env  # Отредактируйте значения
   ```

3. Запустите контейнеры:
   ```bash
   docker-compose up -d
   ```

4. Откройте браузер: `http://localhost` или `https://localhost`

Подробнее см. [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md#docker-deployment)

### В8: Можно ли установить систему без Docker?

Да, система поддерживает ручную установку. Необходимо установить:
- Node.js 18+
- PostgreSQL 14+
- Redis 7+
- Nginx 1.20+

Подробные инструкции см. [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md#manual-deployment)

### В9: Как обновить систему до новой версии?

**С Docker:**
```bash
git pull origin main
docker-compose down
docker-compose pull
docker-compose up -d
```

**Без Docker:**
```bash
git pull origin main
npm install
npm run build
npm run migrate:up
pm2 restart rgsystem-api
```

### В10: Нужно ли мне покупать SSL-сертификат?

Нет, вы можете использовать бесплатные SSL-сертификаты от **Let's Encrypt**:
```bash
certbot --nginx -d rgsystem.example.com
```

Для локальных соревнований можно использовать самоподписанные сертификаты.

---

## Конфигурация / Configuration

### В11: Как изменить порты по умолчанию?

Отредактируйте файл `.env`:
```bash
HTTP_PORT=8080
HTTPS_PORT=8443
API_PORT=3000
```

После изменения перезапустите контейнеры:
```bash
docker-compose down
docker-compose up -d
```

### В12: Как настроить email-уведомления?

Добавьте SMTP настройки в `.env`:
```bash
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_FROM=noreply@rgsystem.local
```

Для Gmail необходимо создать App Password в настройках безопасности.

### В13: Как настроить резервное копирование?

Система автоматически создает резервные копии согласно расписанию в `.env`:
```bash
BACKUP_PATH=/var/backups/rgsystem
BACKUP_RETENTION=30
BACKUP_FULL_SCHEDULE=0 2 * * 0    # Воскресенье 02:00
BACKUP_INCREMENTAL_SCHEDULE=0 3 * * *  # Ежедневно 03:00
```

Для восстановления используйте скрипт:
```bash
./scripts/restore-backup.sh /var/backups/rgsystem/2025-01-15-full.sql.gz
```

### В14: Как ограничить доступ по IP-адресам?

Добавьте правила в `nginx/conf.d/security.conf`:
```nginx
location /api/admin {
    allow 192.168.1.0/24;
    allow 10.0.0.0/8;
    deny all;
}
```

### В15: Как включить двухфакторную аутентификацию (2FA)?

2FA включается в настройках пользователя:
1. Войдите в систему → Профиль → Безопасность
2. Нажмите "Включить 2FA"
3. Отсканируйте QR-код в приложении (Google Authenticator, Authy)
4. Введите 6-значный код для подтверждения

---

## Судейство и оценки / Judging and Scoring

### В16: Сколько судей должно быть на панели?

Согласно правилам FIG 2025-2028:
- **D-панель:** 2-4 судьи (рекомендуется 4)
- **E-панель:** 4-6 судей (рекомендуется 6)
- **A-панель:** 2 судьи (только для групповых упражнений)
- **Линейные судьи:** 4 судьи (по краям ковра)

### В17: Как рассчитывается финальная оценка?

**Формула:**
```
Final Score = D-score + E-score + A-score - ND
```

Где:
- **D-score** = DB + DA + DS + DD (сложность)
- **E-score** = 10.0 - сбавки за технику (среднее после удаления min/max)
- **A-score** = артистизм для групповых упражнений (среднее)
- **ND** = нейтральные сбавки (время, костюм, поведение)

Подробнее см. [SCORING_ALGORITHM.md](./SCORING_ALGORITHM.md)

### В18: Как работает алгоритм разрешения равенства (тайбрейк)?

При равенстве финальных оценок применяется следующий алгоритм:
1. Сравнение **E-score** (выше — лучше)
2. Если равно → сравнение **D-score** (выше — лучше)
3. Если равно → сравнение **A-score** (выше — лучше, только для групп)
4. Если равно → **ex aequo** (разделенное место)

### В19: Можно ли изменить оценку после подтверждения?

Да, но только с разрешения **Главного судьи**:
1. Судья подает запрос на исправление через систему
2. Главный судья проверяет запрос и причину
3. При одобрении оценка разблокируется для редактирования
4. После исправления оценка подтверждается повторно
5. Все изменения логируются в аудит-журнале

### В20: Как судья может увидеть свои предыдущие оценки?

В личном кабинете судьи доступна вкладка **"Мои оценки"**:
- Фильтрация по соревнованию, дате, спортсмену
- Просмотр истории выставленных оценок
- Статистика (среднее, медиана, разброс)
- Сравнение с другими судьями панели

---

## Технические вопросы / Technical Questions

### В21: Какие браузеры поддерживаются?

**Поддерживаемые браузеры:**
- ✅ Google Chrome 100+ (рекомендуется)
- ✅ Mozilla Firefox 95+
- ✅ Microsoft Edge 100+
- ✅ Safari 15+ (macOS, iOS)
- ✅ Opera 85+

**Не поддерживаются:**
- ❌ Internet Explorer (устарел)
- ❌ Устаревшие версии браузеров

### В22: Работает ли система на мобильных устройствах?

Да, интерфейс **адаптивный (responsive)**:
- 📱 Смартфоны (iOS, Android)
- 📱 Планшеты (iPad, Android tablets)
- 💻 Ноутбуки и настольные компьютеры

Для судей рекомендуется использовать планшеты или ноутбуки для удобства ввода.

### В23: Как работает синхронизация в реальном времени?

Система использует **WebSocket** для двунаправленной связи:
```
Judge Panel → WebSocket Server → Redis Pub/Sub → All Connected Clients
```

События синхронизации:
- `score_submitted` — судья выставил оценку
- `score_confirmed` — оценка подтверждена главным судьей
- `performance_status` — изменение статуса выступления
- `athlete_on_carpet` — спортсмен вышел на ковер

### В24: Какова максимальная нагрузка системы?

**Локальное развертывание:**
- 50-100 одновременных пользователей
- 500-1000 выступлений за соревнование

**Облачное развертывание:**
- 500+ одновременных пользователей
- 5000+ выступлений за соревнование

**Производительность API:**
- Чтение: 1000 TPS (transactions per second)
- Запись: 100 TPS

### В25: Где хранятся загруженные файлы?

**Docker:**
- Путь: `/app/uploads` (внутри контейнера)
- Монтируется в: `./uploads` (на хост-машине)

**Без Docker:**
- Путь: `${PROJECT_ROOT}/uploads`

Рекомендуется настроить регулярное резервное копирование этой директории.

---

## Безопасность / Security

### В26: Как защищены пароли в базе данных?

Пароли хешируются с использованием **bcrypt** (cost factor = 12):
```javascript
const hashedPassword = await bcrypt.hash(plainPassword, 12);
```

Пароли **никогда** не хранятся в открытом виде.

### В27: Защищена ли система от SQL-инъекций?

Да, используются **параметризованные запросы** через Knex.js ORM:
```javascript
// ✅ Безопасно
await db('users').where('email', userEmail).first();

// ❌ Небезопасно (не используется)
await db.raw(`SELECT * FROM users WHERE email = '${userEmail}'`);
```

### В28: Как защититься от XSS-атак?

Применяются следующие меры:
- Санитизация пользовательского ввода (DOMPurify)
- CSP (Content Security Policy) заголовки
- Escape HTML в шаблонах
- HttpOnly cookies для JWT токенов

### В29: Истекают ли сессии автоматически?

Да:
- **Access token:** 1 час (настраивается в `JWT_EXPIRES_IN`)
- **Refresh token:** 7 дней (настраивается в `JWT_REFRESH_EXPIRES_IN`)
- При истечении access token, система автоматически запрашивает новый через refresh token

### В30: Логируются ли попытки несанкционированного доступа?

Да, все попытки логируются в `/app/logs/security.log`:
```json
{
  "timestamp": "2025-01-15T14:32:10.123Z",
  "event": "unauthorized_access",
  "ip": "192.168.1.100",
  "endpoint": "/api/admin/users",
  "user_id": null,
  "action": "blocked"
}
```

---

## Производительность / Performance

### В31: Как ускорить работу системы?

**Рекомендации:**
1. Включите Redis кеширование (по умолчанию включено)
2. Используйте CDN для статических файлов
3. Настройте gzip-сжатие в Nginx (включено по умолчанию)
4. Включите HTTP/2 в Nginx
5. Увеличьте пул соединений к БД (`DB_POOL_MAX=20`)

### В32: Что делать, если база данных растет слишком быстро?

1. Настройте автоматическую очистку старых логов:
   ```sql
   DELETE FROM audit_logs WHERE created_at < NOW() - INTERVAL '90 days';
   ```

2. Включите партиционирование таблиц по датам:
   ```sql
   CREATE TABLE audit_logs_2025_01 PARTITION OF audit_logs
   FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
   ```

3. Архивируйте старые соревнования в отдельную БД

### В33: Как мониторить производительность системы?

**Встроенные endpoints:**
- `GET /api/health` — статус здоровья системы
- `GET /api/metrics` — метрики Prometheus (если включено)

**Метрики:**
- Response time (p50, p95, p99)
- Request rate (req/s)
- Error rate (errors/s)
- Database connections
- Memory usage
- CPU usage

Рекомендуется использовать **Grafana + Prometheus** для визуализации.

---

## Устранение неполадок / Troubleshooting

### В34: "Cannot connect to database" — что делать?

**Проверьте:**
1. Работает ли контейнер PostgreSQL:
   ```bash
   docker ps | grep postgres
   ```

2. Правильные ли учетные данные в `.env`:
   ```bash
   DB_HOST=postgres
   DB_PORT=5432
   DB_NAME=rgsystem_db
   DB_USER=rgsystem_user
   DB_PASSWORD=YourPassword
   ```

3. Здоровье базы данных:
   ```bash
   docker exec rgsystem_postgres pg_isready -U rgsystem_user
   ```

### В35: WebSocket соединение не устанавливается

**Решения:**
1. Проверьте, что WebSocket сервер запущен:
   ```bash
   docker logs rgsystem_websocket
   ```

2. Проверьте конфигурацию Nginx для WebSocket:
   ```nginx
   location /ws {
       proxy_http_version 1.1;
       proxy_set_header Upgrade $http_upgrade;
       proxy_set_header Connection "upgrade";
   }
   ```

3. Отключите VPN/прокси, которые могут блокировать WebSocket

### В36: Оценки не сохраняются — в чем причина?

**Возможные причины:**
1. Истек JWT токен → обновите страницу
2. Нет прав доступа → проверьте роль пользователя
3. Выступление уже завершено → попросите главного судью разблокировать
4. Проблема с Redis → проверьте `docker logs rgsystem_redis`

### В37: Как восстановить забытый пароль администратора?

Через консоль базы данных:
```bash
docker exec -it rgsystem_postgres psql -U rgsystem_user -d rgsystem_db
```

```sql
-- Установить новый пароль (bcrypt hash для "NewPassword123!")
UPDATE users
SET password_hash = '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewY5GyYIiXuxJ6cG'
WHERE email = 'admin@example.com';
```

**Внимание:** Измените пароль сразу после входа!

### В38: Ошибка "Port already in use"

Порт занят другим приложением. Измените порт в `.env`:
```bash
HTTP_PORT=8080  # вместо 80
HTTPS_PORT=8443  # вместо 443
```

Или остановите приложение, занимающее порт:
```bash
# Найти процесс
sudo lsof -i :80

# Остановить процесс
sudo kill -9 <PID>
```

### В39: Логи переполняют диск — как очистить?

**Автоматическая ротация:**
```bash
# Настройте logrotate
sudo nano /etc/logrotate.d/rgsystem
```

```
/app/logs/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
}
```

**Ручная очистка:**
```bash
# Очистить логи старше 30 дней
find ./logs -name "*.log" -mtime +30 -delete
```

### В40: Как откатить миграцию базы данных?

```bash
# Просмотр примененных миграций
npm run migrate:status

# Откат последней миграции
npm run migrate:down

# Откат до конкретной версии
npm run migrate:down --to 20250115_create_competitions
```

**Внимание:** Создайте резервную копию перед откатом!

---

## Интеграция и API / Integration and API

### В41: Как получить API токен для интеграции?

1. Войдите в систему как администратор
2. Перейдите в раздел **API Tokens** → **Create New Token**
3. Укажите название и срок действия
4. Сохраните токен (он показывается только один раз!)

Используйте токен в заголовке запроса:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" \
     https://api.rgsystem.local/api/competitions
```

### В42: Есть ли ограничения на количество API запросов?

Да, по умолчанию:
- **1000 запросов в час** на один IP-адрес
- Настраивается в `.env` (`API_RATE_LIMIT`, `API_RATE_WINDOW`)

При превышении лимита сервер вернет:
```json
{
  "error": "Rate limit exceeded",
  "retry_after": 3600
}
```

### В43: Где найти документацию по API?

- **OpenAPI спецификация:** [API_SPECIFICATION.md](./API_SPECIFICATION.md)
- **Interactive Swagger UI:** `https://your-domain/api/docs`
- **Postman коллекция:** `./docs/postman-collection.json`

### В44: Можно ли интегрироваться с внешними системами?

Да, система предоставляет:
- **REST API** для всех операций
- **WebSocket** для real-time уведомлений
- **Webhooks** для событий (создание соревнования, завершение выступления)
- **Export** в форматы: JSON, CSV, Excel, PDF

### В45: Как экспортировать результаты соревнования?

**Через API:**
```bash
GET /api/competitions/{id}/results?format=xlsx
```

**Через интерфейс:**
1. Откройте страницу соревнования
2. Вкладка **Results** → кнопка **Export**
3. Выберите формат: Excel, PDF, CSV, JSON

---

## Лицензирование и поддержка / Licensing and Support

### В46: Какая лицензия у проекта?

Уточните у правообладателя проекта. Документация не содержит информации о лицензии.

### В47: Где получить техническую поддержку?

- **GitHub Issues:** https://github.com/your-org/rgsystem/issues
- **Email:** support@rgsystem.local
- **Документация:** См. все файлы в `/spec_improved/`
- **Community Forum:** https://forum.rgsystem.local

### В48: Как сообщить об ошибке (bug)?

1. Проверьте, что ошибка воспроизводится на последней версии
2. Создайте issue на GitHub с описанием:
   - Шаги для воспроизведения
   - Ожидаемое поведение
   - Фактическое поведение
   - Логи ошибок
   - Версия системы и браузера

### В49: Можно ли запросить новую функцию?

Да! Создайте **Feature Request** на GitHub:
- Опишите желаемую функциональность
- Объясните бизнес-ценность
- Приложите mockup/wireframe (опционально)

### В50: Как внести вклад в проект?

1. Форкните репозиторий
2. Создайте feature ветку: `git checkout -b feature/my-feature`
3. Внесите изменения и добавьте тесты
4. Создайте Pull Request с описанием изменений
5. Дождитесь ревью и одобрения

См. `CONTRIBUTING.md` для подробностей.

---

## Полезные ссылки / Useful Links

### Документация проекта:
- 📖 [README.md](./README.md) — навигация по документации
- 📋 [ANALYSIS_REPORT.md](./ANALYSIS_REPORT.md) — анализ требований
- 📚 [GLOSSARY.md](./GLOSSARY.md) — терминология FIG
- 🗄️ [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) — схема базы данных
- 🔌 [API_SPECIFICATION.md](./API_SPECIFICATION.md) — REST API
- 🧮 [SCORING_ALGORITHM.md](./SCORING_ALGORITHM.md) — алгоритмы подсчета
- 🚀 [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md) — установка и развертывание
- ✅ [TEST_CASES.md](./TEST_CASES.md) — тестовые случаи

### Внешние ресурсы:
- 🏅 [FIG Official Website](https://www.gymnastics.sport/)
- 📘 [FIG Code of Points 2025-2028](https://www.gymnastics.sport/site/rules/)
- 🐳 [Docker Documentation](https://docs.docker.com/)
- 🐘 [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

## Обратная связь / Feedback

Если вы не нашли ответ на свой вопрос, пожалуйста:
- Создайте issue на GitHub
- Напишите на support@rgsystem.local
- Предложите дополнения к FAQ через Pull Request

**Последнее обновление:** 2025-01-15
