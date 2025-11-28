# Спецификация обработки ошибок
# Error Handling Specification

## Содержание / Table of Contents

1. [Введение](#введение--introduction)
2. [Классификация ошибок](#классификация-ошибок--error-classification)
3. [Коды ошибок](#коды-ошибок--error-codes)
4. [Структура ответов об ошибках](#структура-ответов-об-ошибках--error-response-structure)
5. [Обработка ошибок по слоям](#обработка-ошибок-по-слоям--layer-specific-error-handling)
6. [Стратегии восстановления](#стратегии-восстановления--recovery-strategies)
7. [Логирование ошибок](#логирование-ошибок--error-logging)
8. [Пользовательские сообщения](#пользовательские-сообщения--user-facing-messages)
9. [Мониторинг и алертинг](#мониторинг-и-алертинг--monitoring-and-alerting)

---

## Введение / Introduction

Данный документ описывает единый подход к обработке ошибок в системе RG System на всех уровнях: от пользовательского интерфейса до базы данных.

### Цели обработки ошибок:
- ✅ Предоставить пользователю понятную информацию об ошибке
- ✅ Логировать детальную техническую информацию для разработчиков
- ✅ Обеспечить graceful degradation (плавную деградацию)
- ✅ Предотвратить утечку чувствительной информации
- ✅ Поддерживать восстановление после сбоев

---

## Классификация ошибок / Error Classification

### 1. По источнику ошибки

#### Пользовательские ошибки (User Errors)
Ошибки, вызванные некорректными действиями пользователя.

**Примеры:**
- Неверный формат email
- Пустые обязательные поля
- Недостаточно прав доступа
- Превышен лимит запросов

**Обработка:**
- Показать дружелюбное сообщение пользователю
- Не логировать как критическую ошибку
- Подсветить проблемное поле в форме

#### Системные ошибки (System Errors)
Ошибки, вызванные сбоями в работе системы.

**Примеры:**
- Потеря соединения с БД
- Ошибка Redis
- Недостаток памяти
- Ошибка файловой системы

**Обработка:**
- Логировать с уровнем ERROR
- Отправить alert администраторам
- Показать общее сообщение пользователю
- Активировать механизм восстановления

#### Внешние ошибки (External Errors)
Ошибки при взаимодействии с внешними сервисами.

**Примеры:**
- Timeout при вызове API
- Ошибка SMTP сервера
- CDN недоступен
- Ошибка платежного шлюза

**Обработка:**
- Реализовать retry с exponential backoff
- Использовать fallback механизмы
- Логировать для анализа
- Показать пользователю альтернативные действия

#### Бизнес-логические ошибки (Business Logic Errors)
Ошибки, связанные с нарушением бизнес-правил.

**Примеры:**
- Попытка удалить активное соревнование
- Оценка вне допустимого диапазона
- Превышено максимальное количество судей
- Дублирование регистрационного номера

**Обработка:**
- Показать конкретное объяснение нарушенного правила
- Предложить корректирующие действия
- Логировать для аудита

### 2. По критичности

| Уровень | Описание | Действия |
|---------|----------|----------|
| **CRITICAL** | Полный отказ системы | Alert → On-call инженер, автоматический failover |
| **ERROR** | Ошибка в выполнении операции | Логирование, уведомление администратора |
| **WARNING** | Потенциальная проблема | Логирование для анализа |
| **INFO** | Информационное событие | Логирование для аудита |

### 3. По возможности восстановления

#### Recoverable (Восстанавливаемые)
Ошибки, после которых система может продолжить работу.

**Примеры:**
- Временная потеря WebSocket соединения → автоматическое переподключение
- Timeout запроса → повторная попытка
- Валидация формы → исправление пользователем

#### Non-recoverable (Невосстанавливаемые)
Критические ошибки, требующие вмешательства.

**Примеры:**
- Полный отказ БД → требуется восстановление из backup
- Критическая ошибка в алгоритме подсчета → hotfix
- Переполнение диска → расширение хранилища

---

## Коды ошибок / Error Codes

### HTTP статус-коды

| Код | Название | Когда использовать |
|-----|----------|-------------------|
| **200** | OK | Успешный запрос |
| **201** | Created | Ресурс успешно создан |
| **204** | No Content | Успешное удаление |
| **400** | Bad Request | Невалидные данные от клиента |
| **401** | Unauthorized | Отсутствует или невалиден токен |
| **403** | Forbidden | Нет прав доступа |
| **404** | Not Found | Ресурс не найден |
| **409** | Conflict | Конфликт (дубликат, состояние) |
| **422** | Unprocessable Entity | Валидация не пройдена |
| **429** | Too Many Requests | Превышен rate limit |
| **500** | Internal Server Error | Внутренняя ошибка сервера |
| **502** | Bad Gateway | Ошибка внешнего сервиса |
| **503** | Service Unavailable | Сервис временно недоступен |
| **504** | Gateway Timeout | Timeout внешнего сервиса |

### Пользовательские коды ошибок

Формат: `RG-[MODULE]-[TYPE]-[NUMBER]`

#### Модули (MODULE)
- `AUTH` — аутентификация и авторизация
- `COMP` — управление соревнованиями
- `JUDGE` — судейство и оценки
- `SYNC` — синхронизация данных
- `DB` — операции с базой данных
- `CACHE` — операции с кешем
- `FILE` — работа с файлами
- `API` — REST API

#### Типы (TYPE)
- `VAL` — ошибка валидации
- `PERM` — ошибка прав доступа
- `STATE` — ошибка состояния
- `CONN` — ошибка соединения
- `CALC` — ошибка расчета
- `SYS` — системная ошибка

#### Примеры кодов

```javascript
const ERROR_CODES = {
  // Аутентификация
  'RG-AUTH-VAL-001': 'Invalid email format',
  'RG-AUTH-VAL-002': 'Password too weak',
  'RG-AUTH-PERM-001': 'Invalid credentials',
  'RG-AUTH-PERM-002': 'Account locked',
  'RG-AUTH-STATE-001': 'Token expired',
  'RG-AUTH-STATE-002': '2FA code invalid',

  // Соревнования
  'RG-COMP-VAL-001': 'Competition name is required',
  'RG-COMP-VAL-002': 'Start date must be before end date',
  'RG-COMP-STATE-001': 'Cannot delete active competition',
  'RG-COMP-STATE-002': 'Competition already finalized',
  'RG-COMP-PERM-001': 'Only organizer can modify competition',

  // Судейство
  'RG-JUDGE-VAL-001': 'Score out of valid range',
  'RG-JUDGE-VAL-002': 'Insufficient judges on panel',
  'RG-JUDGE-STATE-001': 'Score already confirmed',
  'RG-JUDGE-STATE-002': 'Performance not started',
  'RG-JUDGE-CALC-001': 'Error calculating final score',

  // Синхронизация
  'RG-SYNC-CONN-001': 'WebSocket connection failed',
  'RG-SYNC-CONN-002': 'Redis pub/sub error',
  'RG-SYNC-STATE-001': 'Sync conflict detected',

  // База данных
  'RG-DB-CONN-001': 'Database connection lost',
  'RG-DB-SYS-001': 'Query timeout',
  'RG-DB-VAL-001': 'Unique constraint violation',
  'RG-DB-VAL-002': 'Foreign key constraint violation',

  // Кеширование
  'RG-CACHE-CONN-001': 'Redis connection failed',
  'RG-CACHE-SYS-001': 'Cache eviction error',

  // Файлы
  'RG-FILE-VAL-001': 'File size exceeds limit',
  'RG-FILE-VAL-002': 'Invalid file type',
  'RG-FILE-SYS-001': 'File system error',

  // API
  'RG-API-VAL-001': 'Invalid request parameters',
  'RG-API-PERM-001': 'API rate limit exceeded',
  'RG-API-SYS-001': 'Internal server error',
};
```

---

## Структура ответов об ошибках / Error Response Structure

### Стандартный формат ответа

```typescript
interface ErrorResponse {
  /** HTTP статус код */
  status: number;

  /** Пользовательский код ошибки */
  code: string;

  /** Сообщение для пользователя (локализованное) */
  message: string;

  /** Детали ошибок (для валидации форм) */
  details?: ValidationError[];

  /** Путь к ресурсу, где произошла ошибка */
  path?: string;

  /** Timestamp ошибки */
  timestamp: string;

  /** Trace ID для отслеживания в логах */
  trace_id: string;

  /** Дополнительная информация (только в development) */
  debug?: {
    stack?: string;
    query?: string;
    params?: Record<string, any>;
  };
}

interface ValidationError {
  /** Поле, в котором ошибка */
  field: string;

  /** Код ошибки валидации */
  code: string;

  /** Сообщение об ошибке */
  message: string;

  /** Полученное значение */
  value?: any;
}
```

### Примеры ответов

#### 400 Bad Request (валидация)

```json
{
  "status": 400,
  "code": "RG-COMP-VAL-001",
  "message": "Validation failed",
  "details": [
    {
      "field": "name",
      "code": "required",
      "message": "Competition name is required"
    },
    {
      "field": "date_start",
      "code": "invalid_date",
      "message": "Start date must be in the future",
      "value": "2024-01-01"
    }
  ],
  "path": "/api/competitions",
  "timestamp": "2025-01-15T14:30:00.000Z",
  "trace_id": "a7b3c4d5-e6f7-8901-2345-6789abcdef01"
}
```

#### 401 Unauthorized

```json
{
  "status": 401,
  "code": "RG-AUTH-STATE-001",
  "message": "Your session has expired. Please log in again.",
  "path": "/api/competitions/123",
  "timestamp": "2025-01-15T14:30:00.000Z",
  "trace_id": "b8c4d5e6-f7a8-9012-3456-789abcdef012"
}
```

#### 403 Forbidden

```json
{
  "status": 403,
  "code": "RG-COMP-PERM-001",
  "message": "You don't have permission to modify this competition. Only the organizer can make changes.",
  "path": "/api/competitions/123",
  "timestamp": "2025-01-15T14:30:00.000Z",
  "trace_id": "c9d5e6f7-a8b9-0123-4567-89abcdef0123"
}
```

#### 409 Conflict

```json
{
  "status": 409,
  "code": "RG-JUDGE-STATE-001",
  "message": "Score has already been confirmed and cannot be modified. Please contact the Chief Judge to unlock.",
  "path": "/api/scores/456",
  "timestamp": "2025-01-15T14:30:00.000Z",
  "trace_id": "d0e6f7a8-b9c0-1234-5678-9abcdef01234"
}
```

#### 500 Internal Server Error

```json
{
  "status": 500,
  "code": "RG-API-SYS-001",
  "message": "An unexpected error occurred. Our team has been notified and is working to fix it.",
  "path": "/api/scores/calculate",
  "timestamp": "2025-01-15T14:30:00.000Z",
  "trace_id": "e1f7a8b9-c0d1-2345-6789-abcdef012345"
}
```

---

## Обработка ошибок по слоям / Layer-Specific Error Handling

### 1. Слой базы данных (Database Layer)

```javascript
// database/connection.js
const db = knex({
  client: 'postgresql',
  connection: {
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    database: process.env.DB_NAME,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
  },
  pool: {
    min: 2,
    max: 10,
    // Обработка ошибок пула соединений
    afterCreate: (conn, done) => {
      conn.on('error', (err) => {
        logger.error('Database connection error', {
          error: err,
          code: 'RG-DB-CONN-001',
        });
        // Попытка переподключения
        reconnectDatabase();
      });
      done();
    },
  },
});

// Retry logic для транзакционных операций
async function executeWithRetry(operation, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      // Ошибки, которые можно повторить
      const retryableCodes = ['ECONNRESET', 'ETIMEDOUT', '40001']; // 40001 = serialization failure

      if (retryableCodes.includes(error.code) && attempt < maxRetries) {
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        logger.warn(`Retrying database operation (attempt ${attempt}/${maxRetries})`, {
          delay,
          error: error.message,
        });
        await sleep(delay);
        continue;
      }

      throw error;
    }
  }
}

// Обработка специфичных ошибок PostgreSQL
function handleDatabaseError(error) {
  if (error.code === '23505') {
    // Unique constraint violation
    return {
      status: 409,
      code: 'RG-DB-VAL-001',
      message: 'This record already exists',
      details: [{
        field: error.constraint.replace(/_unique$/, ''),
        code: 'duplicate',
        message: 'Value must be unique',
      }],
    };
  }

  if (error.code === '23503') {
    // Foreign key constraint violation
    return {
      status: 400,
      code: 'RG-DB-VAL-002',
      message: 'Referenced record does not exist',
    };
  }

  if (error.code === '57014') {
    // Query timeout
    return {
      status: 504,
      code: 'RG-DB-SYS-001',
      message: 'Database query timeout. Please try again.',
    };
  }

  // Общая ошибка БД
  return {
    status: 500,
    code: 'RG-DB-SYS-002',
    message: 'Database error occurred',
  };
}
```

### 2. Слой бизнес-логики (Service Layer)

```javascript
// services/competitionService.js
class CompetitionService {
  async createCompetition(data) {
    // Валидация бизнес-правил
    if (new Date(data.date_start) >= new Date(data.date_end)) {
      throw new BusinessError('RG-COMP-VAL-002',
        'Start date must be before end date', 400);
    }

    // Проверка прав
    if (!user.hasRole('organizer')) {
      throw new PermissionError('RG-COMP-PERM-001',
        'Only organizers can create competitions', 403);
    }

    try {
      return await db.transaction(async (trx) => {
        const competition = await trx('competitions').insert(data).returning('*');

        // Автоматическое создание связанных структур
        await trx('competition_settings').insert({
          competition_id: competition[0].id,
          ...defaultSettings,
        });

        return competition[0];
      });
    } catch (error) {
      logger.error('Failed to create competition', {
        error,
        data,
        user_id: user.id,
      });
      throw handleDatabaseError(error);
    }
  }

  async finalizeCompetition(competitionId) {
    const competition = await this.getCompetition(competitionId);

    // Проверка состояния
    if (competition.status === 'finalized') {
      throw new StateError('RG-COMP-STATE-002',
        'Competition is already finalized', 409);
    }

    // Проверка, что все выступления оценены
    const pendingPerformances = await db('performances')
      .where({ competition_id: competitionId, status: 'pending' })
      .count();

    if (pendingPerformances[0].count > 0) {
      throw new StateError('RG-COMP-STATE-003',
        `Cannot finalize: ${pendingPerformances[0].count} performances are not yet scored`, 409);
    }

    // Finalize
    return await db('competitions')
      .where({ id: competitionId })
      .update({ status: 'finalized', finalized_at: new Date() })
      .returning('*');
  }
}

// Кастомные классы ошибок
class BusinessError extends Error {
  constructor(code, message, status = 400) {
    super(message);
    this.code = code;
    this.status = status;
    this.name = 'BusinessError';
  }
}

class PermissionError extends Error {
  constructor(code, message, status = 403) {
    super(message);
    this.code = code;
    this.status = status;
    this.name = 'PermissionError';
  }
}

class StateError extends Error {
  constructor(code, message, status = 409) {
    super(message);
    this.code = code;
    this.status = status;
    this.name = 'StateError';
  }
}
```

### 3. Слой API (Controller Layer)

```javascript
// controllers/competitionController.js
class CompetitionController {
  async create(req, res, next) {
    try {
      // Валидация входных данных
      const validatedData = await validateCompetitionData(req.body);

      // Создание соревнования
      const competition = await competitionService.createCompetition(
        validatedData,
        req.user
      );

      res.status(201).json({
        success: true,
        data: competition,
      });
    } catch (error) {
      next(error); // Передача в error middleware
    }
  }
}

// Middleware для обработки ошибок
function errorHandler(error, req, res, next) {
  // Генерация trace ID для отслеживания
  const traceId = generateTraceId();

  // Логирование ошибки
  logger.error('Request error', {
    trace_id: traceId,
    error: {
      name: error.name,
      message: error.message,
      stack: error.stack,
      code: error.code,
    },
    request: {
      method: req.method,
      path: req.path,
      params: req.params,
      query: req.query,
      body: sanitizeBody(req.body), // Удаление паролей и токенов
      user_id: req.user?.id,
      ip: req.ip,
    },
  });

  // Определение статуса и сообщения
  let status = error.status || 500;
  let code = error.code || 'RG-API-SYS-001';
  let message = error.message || 'Internal server error';
  let details = error.details || undefined;

  // Обработка ошибок валидации (Joi, Yup)
  if (error.name === 'ValidationError') {
    status = 400;
    code = 'RG-API-VAL-001';
    message = 'Validation failed';
    details = error.details.map(d => ({
      field: d.path[0],
      code: d.type,
      message: d.message,
      value: d.context?.value,
    }));
  }

  // Обработка ошибок JWT
  if (error.name === 'JsonWebTokenError') {
    status = 401;
    code = 'RG-AUTH-STATE-001';
    message = 'Invalid authentication token';
  }

  if (error.name === 'TokenExpiredError') {
    status = 401;
    code = 'RG-AUTH-STATE-001';
    message = 'Your session has expired. Please log in again.';
  }

  // Не показывать внутренние ошибки пользователям
  if (status === 500) {
    message = 'An unexpected error occurred. Our team has been notified.';
  }

  // Формирование ответа
  const response = {
    status,
    code,
    message,
    details,
    path: req.path,
    timestamp: new Date().toISOString(),
    trace_id: traceId,
  };

  // Добавление debug информации в development
  if (process.env.NODE_ENV === 'development') {
    response.debug = {
      stack: error.stack,
      originalMessage: error.message,
    };
  }

  res.status(status).json(response);
}
```

### 4. Слой WebSocket

```javascript
// websocket/errorHandler.js
class WebSocketErrorHandler {
  handleConnectionError(socket, error) {
    logger.error('WebSocket connection error', {
      socket_id: socket.id,
      user_id: socket.user?.id,
      error,
      code: 'RG-SYNC-CONN-001',
    });

    // Отправка сообщения об ошибке клиенту
    socket.emit('error', {
      code: 'RG-SYNC-CONN-001',
      message: 'Connection error occurred. Attempting to reconnect...',
      recoverable: true,
    });

    // Попытка переподключения на клиенте
    // (обрабатывается автоматически Socket.IO)
  }

  handleScoreSubmissionError(socket, error, scoreData) {
    logger.error('Score submission error', {
      socket_id: socket.id,
      user_id: socket.user?.id,
      score_data: scoreData,
      error,
      code: 'RG-JUDGE-VAL-001',
    });

    // Уведомление судьи об ошибке
    socket.emit('score_submission_failed', {
      code: error.code || 'RG-JUDGE-SYS-001',
      message: error.message || 'Failed to submit score',
      performance_id: scoreData.performance_id,
      recoverable: true,
    });
  }

  handleSyncConflict(socket, conflict) {
    logger.warn('Sync conflict detected', {
      socket_id: socket.id,
      user_id: socket.user?.id,
      conflict,
      code: 'RG-SYNC-STATE-001',
    });

    // Запрос разрешения конфликта от клиента
    socket.emit('sync_conflict', {
      code: 'RG-SYNC-STATE-001',
      message: 'Data conflict detected. Please refresh.',
      local_version: conflict.local_version,
      server_version: conflict.server_version,
      action_required: 'refresh',
    });
  }
}
```

### 5. Фронтенд (Frontend Layer)

```javascript
// frontend/utils/errorHandler.js
class FrontendErrorHandler {
  handleApiError(error, context) {
    const response = error.response?.data;

    if (!response) {
      // Сетевая ошибка (нет соединения)
      this.showNetworkError();
      return;
    }

    switch (response.status) {
      case 400:
        // Валидация - показать ошибки под полями
        this.showValidationErrors(response.details, context.form);
        break;

      case 401:
        // Неавторизован - редирект на логин
        this.handleUnauthorized();
        break;

      case 403:
        // Нет прав - показать сообщение
        this.showPermissionDenied(response.message);
        break;

      case 404:
        // Не найдено
        this.showNotFound(response.message);
        break;

      case 409:
        // Конфликт - показать с возможностью действий
        this.showConflict(response);
        break;

      case 429:
        // Rate limit - показать таймер
        this.showRateLimitError(response);
        break;

      case 500:
      case 502:
      case 503:
      case 504:
        // Серверная ошибка
        this.showServerError(response);
        break;

      default:
        this.showGenericError(response.message);
    }

    // Логирование на клиенте для мониторинга
    this.logError(response, context);
  }

  showValidationErrors(details, form) {
    details?.forEach(error => {
      const field = form.querySelector(`[name="${error.field}"]`);
      if (field) {
        field.classList.add('error');
        const errorEl = document.createElement('span');
        errorEl.className = 'field-error';
        errorEl.textContent = error.message;
        field.parentElement.appendChild(errorEl);
      }
    });
  }

  showNetworkError() {
    this.showToast({
      type: 'error',
      title: 'No connection',
      message: 'Please check your internet connection and try again.',
      action: {
        label: 'Retry',
        handler: () => window.location.reload(),
      },
    });
  }

  handleUnauthorized() {
    // Очистка токена
    localStorage.removeItem('access_token');

    // Редирект на страницу логина
    window.location.href = '/login?redirect=' + encodeURIComponent(window.location.pathname);
  }

  showPermissionDenied(message) {
    this.showModal({
      type: 'error',
      title: 'Access Denied',
      message: message || 'You don\'t have permission to perform this action.',
      buttons: [
        { label: 'OK', primary: true },
      ],
    });
  }

  logError(response, context) {
    // Отправка ошибки на сервер для мониторинга
    fetch('/api/client-errors', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        trace_id: response.trace_id,
        error_code: response.code,
        url: window.location.href,
        user_agent: navigator.userAgent,
        context,
      }),
    }).catch(() => {
      // Игнорируем ошибки логирования
    });
  }
}
```

---

## Стратегии восстановления / Recovery Strategies

### 1. Автоматическое переподключение (Auto-reconnect)

```javascript
// frontend/services/websocket.js
class WebSocketService {
  constructor() {
    this.reconnectAttempts = 0;
    this.maxReconnectAttempts = 5;
    this.reconnectDelay = 1000; // Начальная задержка 1 сек
  }

  connect() {
    this.socket = io(WS_URL, {
      auth: { token: getAuthToken() },
      reconnection: true,
      reconnectionAttempts: this.maxReconnectAttempts,
      reconnectionDelay: this.reconnectDelay,
      reconnectionDelayMax: 10000, // Максимум 10 сек
      timeout: 20000,
    });

    this.socket.on('connect_error', (error) => {
      this.reconnectAttempts++;
      logger.warn('WebSocket connection failed', {
        attempt: this.reconnectAttempts,
        max_attempts: this.maxReconnectAttempts,
        error: error.message,
      });

      if (this.reconnectAttempts >= this.maxReconnectAttempts) {
        this.showReconnectionFailed();
      } else {
        this.showReconnecting(this.reconnectAttempts);
      }
    });

    this.socket.on('connect', () => {
      this.reconnectAttempts = 0;
      logger.info('WebSocket connected');
      this.hideReconnecting();
    });
  }

  showReconnecting(attempt) {
    showBanner({
      type: 'warning',
      message: `Reconnecting to server... (${attempt}/${this.maxReconnectAttempts})`,
      persistent: true,
    });
  }

  showReconnectionFailed() {
    showBanner({
      type: 'error',
      message: 'Unable to connect to server. Please refresh the page.',
      action: {
        label: 'Refresh',
        handler: () => window.location.reload(),
      },
      persistent: true,
    });
  }
}
```

### 2. Retry с Exponential Backoff

```javascript
// utils/retry.js
async function retryWithBackoff(operation, options = {}) {
  const {
    maxAttempts = 3,
    initialDelay = 1000,
    maxDelay = 10000,
    backoffFactor = 2,
    retryableErrors = ['ECONNRESET', 'ETIMEDOUT', 'ENOTFOUND'],
  } = options;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error) {
      const isLastAttempt = attempt === maxAttempts;
      const isRetryable = retryableErrors.includes(error.code) ||
                          error.status >= 500;

      if (isLastAttempt || !isRetryable) {
        throw error;
      }

      const delay = Math.min(
        initialDelay * Math.pow(backoffFactor, attempt - 1),
        maxDelay
      );

      logger.warn(`Retrying operation (attempt ${attempt}/${maxAttempts})`, {
        delay,
        error: error.message,
      });

      await sleep(delay);
    }
  }
}

// Использование
const result = await retryWithBackoff(
  () => fetch('/api/scores/calculate'),
  { maxAttempts: 3, initialDelay: 1000 }
);
```

### 3. Circuit Breaker

```javascript
// utils/circuitBreaker.js
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 60000; // 1 минута

    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failures = 0;
    this.successes = 0;
    this.nextAttempt = Date.now();
  }

  async execute(operation) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      // Переход в HALF_OPEN для проверки
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await operation();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;

    if (this.state === 'HALF_OPEN') {
      this.successes++;
      if (this.successes >= this.successThreshold) {
        this.state = 'CLOSED';
        this.successes = 0;
        logger.info('Circuit breaker closed');
      }
    }
  }

  onFailure() {
    this.failures++;

    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
      logger.error('Circuit breaker opened', {
        failures: this.failures,
        next_attempt: new Date(this.nextAttempt),
      });
    }
  }
}

// Использование для внешнего API
const externalApiBreaker = new CircuitBreaker({
  failureThreshold: 5,
  timeout: 60000,
});

async function callExternalAPI(data) {
  return await externalApiBreaker.execute(() =>
    fetch('https://external-api.com/endpoint', {
      method: 'POST',
      body: JSON.stringify(data),
    })
  );
}
```

### 4. Graceful Degradation

```javascript
// services/scoringService.js
class ScoringService {
  async calculateFinalScore(performanceId) {
    try {
      // Попытка рассчитать с использованием кеша
      const cachedResult = await cache.get(`score:${performanceId}`);
      if (cachedResult) {
        return cachedResult;
      }

      // Расчет с базы данных
      const result = await this.calculateFromDatabase(performanceId);

      // Кеширование результата
      await cache.set(`score:${performanceId}`, result, 3600);

      return result;
    } catch (error) {
      logger.error('Score calculation failed', { performanceId, error });

      // Fallback: попытка получить из базы без кеша
      try {
        return await this.calculateFromDatabase(performanceId);
      } catch (dbError) {
        // Последний fallback: вернуть предыдущий результат
        const previousScore = await this.getPreviousScore(performanceId);
        if (previousScore) {
          logger.warn('Returning previous score due to calculation error', {
            performanceId,
          });
          return { ...previousScore, is_stale: true };
        }

        throw dbError;
      }
    }
  }
}
```

---

## Логирование ошибок / Error Logging

### Уровни логирования

```javascript
// config/logger.js
const winston = require('winston');

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'rgsystem-api',
    environment: process.env.NODE_ENV,
  },
  transports: [
    // Все ошибки ERROR и выше в отдельный файл
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 10485760, // 10MB
      maxFiles: 10,
    }),

    // Все логи в общий файл
    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 10485760,
      maxFiles: 30,
    }),

    // Вывод в консоль (только в development)
    ...(process.env.NODE_ENV === 'development' ? [
      new winston.transports.Console({
        format: winston.format.combine(
          winston.format.colorize(),
          winston.format.simple()
        ),
      }),
    ] : []),
  ],
});
```

### Структура лог-записей

```json
{
  "timestamp": "2025-01-15T14:30:00.123Z",
  "level": "error",
  "message": "Failed to calculate final score",
  "service": "rgsystem-api",
  "environment": "production",
  "trace_id": "a7b3c4d5-e6f7-8901-2345-6789abcdef01",
  "error": {
    "name": "CalculationError",
    "message": "Insufficient E-scores",
    "code": "RG-JUDGE-CALC-001",
    "stack": "Error: Insufficient E-scores\n    at ScoringService.calculateFinalScore..."
  },
  "context": {
    "performance_id": "123e4567-e89b-12d3-a456-426614174000",
    "user_id": "789e4567-e89b-12d3-a456-426614174001",
    "competition_id": "456e4567-e89b-12d3-a456-426614174002"
  },
  "request": {
    "method": "POST",
    "path": "/api/scores/calculate",
    "ip": "192.168.1.100",
    "user_agent": "Mozilla/5.0..."
  }
}
```

---

## Пользовательские сообщения / User-Facing Messages

### Принципы написания сообщений об ошибках

1. **Ясность**: Объясните, что пошло не так
2. **Контекст**: Где произошла ошибка
3. **Действие**: Что пользователь может сделать
4. **Тон**: Дружелюбный и помогающий

### Примеры хороших сообщений

❌ **Плохо:**
```
Error: 23505
```

✅ **Хорошо:**
```
This competition name is already taken. Please choose a different name.
```

---

❌ **Плохо:**
```
Forbidden
```

✅ **Хорошо:**
```
You don't have permission to modify this competition. Only the organizer can make changes.
```

---

❌ **Плохо:**
```
Internal server error
```

✅ **Хорошо:**
```
Something went wrong on our end. We've been notified and are working to fix it. Please try again in a few minutes.
```

### Локализация сообщений

```javascript
// locales/ru.json
{
  "errors": {
    "RG-AUTH-VAL-001": "Неверный формат email",
    "RG-AUTH-VAL-002": "Пароль слишком простой. Используйте минимум 8 символов, включая цифры и спецсимволы.",
    "RG-COMP-STATE-001": "Невозможно удалить активное соревнование. Сначала завершите его.",
    "RG-JUDGE-VAL-001": "Оценка вне допустимого диапазона (0.0 - 10.0)"
  }
}

// locales/en.json
{
  "errors": {
    "RG-AUTH-VAL-001": "Invalid email format",
    "RG-AUTH-VAL-002": "Password too weak. Use at least 8 characters with numbers and special characters.",
    "RG-COMP-STATE-001": "Cannot delete an active competition. Please finalize it first.",
    "RG-JUDGE-VAL-001": "Score out of valid range (0.0 - 10.0)"
  }
}
```

---

## Мониторинг и алертинг / Monitoring and Alerting

### Метрики для отслеживания

1. **Error Rate** — процент ошибочных запросов
2. **Error Count by Code** — количество ошибок по кодам
3. **Response Time** — время ответа API
4. **Database Connection Errors** — ошибки соединения с БД
5. **WebSocket Disconnects** — отключения WebSocket
6. **Failed Login Attempts** — неудачные попытки входа

### Правила алертинга

```yaml
# alerts.yml
groups:
  - name: rgsystem_errors
    interval: 60s
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} (threshold: 5%)"

      - alert: DatabaseConnectionFailed
        expr: database_connections_failed_total > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Database connection failed"
          description: "Unable to connect to database"

      - alert: TooManyFailedLogins
        expr: rate(auth_failed_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Suspicious login activity"
          description: "{{ $value }} failed login attempts per second"
```

---

## Заключение / Conclusion

Правильная обработка ошибок — ключевой аспект надежности системы. Следуя этой спецификации, вы обеспечите:

- ✅ Понятные сообщения для пользователей
- ✅ Детальную информацию для отладки
- ✅ Автоматическое восстановление после сбоев
- ✅ Защиту от утечки чувствительных данных
- ✅ Возможность мониторинга и алертинга

**См. также:**
- [API_SPECIFICATION.md](./API_SPECIFICATION.md) — коды ответов API
- [DEPLOYMENT_GUIDE.md](./DEPLOYMENT_GUIDE.md) — troubleshooting
- [FAQ.md](./FAQ.md) — часто задаваемые вопросы
- [TEST_CASES.md](./TEST_CASES.md) — тестирование обработки ошибок
