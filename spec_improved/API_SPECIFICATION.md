# API Спецификация
## Система управления соревнованиями по художественной гимнастике

> **Версия API:** v1.0
> **Дата:** 2025-11-26
> **Базовый URL:** `https://api.rgsystem.example/v1`
> **Протокол:** REST API + WebSocket

---

## Содержание

1. [Аутентификация](#аутентификация)
2. [Competitions API](#competitions-api)
3. [Athletes API](#athletes-api)
4. [Judges API](#judges-api)
5. [Scores API](#scores-api)
6. [Start Lists API](#start-lists-api)
7. [Events API](#events-api)
8. [WebSocket API](#websocket-api)
9. [Коды ошибок](#коды-ошибок)
10. [Примеры использования](#примеры-использования)

---

## Общая информация

### Формат данных
- **Request:** `application/json`
- **Response:** `application/json`
- **Кодировка:** UTF-8
- **Даты:** ISO 8601 (`2025-03-15T14:30:00Z`)

### Пагинация
```json
{
  "data": [...],
  "meta": {
    "total": 150,
    "per_page": 20,
    "current_page": 1,
    "last_page": 8
  }
}
```

### Стандартный формат ответа

**Успешный ответ:**
```json
{
  "success": true,
  "data": { ... }
}
```

**Ответ с ошибкой:**
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  }
}
```

---

## Аутентификация

### POST /auth/login

**Описание:** Вход в систему

**Request:**
```json
{
  "email": "judge@example.com",
  "password": "securePassword123"
}
```

**Response:** `200 OK`
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "expires_in": 3600,
    "token_type": "Bearer",
    "user": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "judge@example.com",
      "first_name": "Мария",
      "last_name": "Иванова",
      "role": "judge"
    }
  }
}
```

---

### POST /auth/refresh

**Описание:** Обновление токена доступа

**Headers:**
```
Authorization: Bearer {refresh_token}
```

**Response:** `200 OK`
```json
{
  "success": true,
  "data": {
    "access_token": "newAccessToken...",
    "expires_in": 3600
  }
}
```

---

### POST /auth/logout

**Описание:** Выход из системы

**Headers:**
```
Authorization: Bearer {access_token}
```

**Response:** `204 No Content`

---

## Competitions API

### GET /competitions

**Описание:** Получить список соревнований

**Query Parameters:**
- `page` (int, default: 1) — номер страницы
- `per_page` (int, default: 20) — элементов на странице
- `status` (string) — фильтр по статусу: `draft`, `active`, `completed`, `archived`
- `date_from` (date) — дата начала (YYYY-MM-DD)
- `date_to` (date) — дата окончания (YYYY-MM-DD)
- `search` (string) — поиск по названию/городу

**Response:** `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "Открытое первенство Москвы",
      "city": "Москва",
      "venue": "СК \"Олимпийский\"",
      "date_start": "2025-04-15",
      "date_end": "2025-04-18",
      "status": "active",
      "athletes_count": 156,
      "groups_count": 48
    }
  ],
  "meta": {
    "total": 12,
    "per_page": 20,
    "current_page": 1,
    "last_page": 1
  }
}
```

---

### GET /competitions/{id}

**Описание:** Получить детальную информацию о соревновании

**Response:** `200 OK`
```json
{
  "success": true,
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Открытое первенство Москвы",
    "city": "Москва",
    "venue": "СК \"Олимпийский\"",
    "address": "Олимпийский проспект, 16",
    "date_start": "2025-04-15",
    "date_end": "2025-04-18",
    "organizer": "Федерация художественной гимнастики Москвы",
    "chief_judge": {
      "id": "judge-uuid",
      "first_name": "Мария",
      "last_name": "Иванова"
    },
    "settings": {
      "individual": {
        "routine_time": 90,
        "warmup_time": 30
      },
      "group": {
        "routine_time": 150,
        "warmup_time": 60
      }
    },
    "status": "active",
    "created_at": "2025-03-01T10:00:00Z",
    "updated_at": "2025-04-15T08:00:00Z"
  }
}
```

---

### POST /competitions

**Описание:** Создать новое соревнование

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request:**
```json
{
  "name": "Кубок России 2025",
  "city": "Санкт-Петербург",
  "venue": "Дворец спорта \"Юбилейный\"",
  "address": "проспект Добролюбова, 18",
  "date_start": "2025-06-10",
  "date_end": "2025-06-13",
  "organizer": "Федерация художественной гимнастики России",
  "categories": ["senior", "junior"],
  "contact": {
    "email": "info@fgr.ru",
    "phone": "+7 (812) 123-45-67",
    "website": "https://fgr.ru"
  }
}
```

**Response:** `201 Created`
```json
{
  "success": true,
  "data": {
    "id": "new-competition-uuid",
    "name": "Кубок России 2025",
    "status": "draft",
    "created_at": "2025-03-20T14:30:00Z"
  }
}
```

---

### PUT /competitions/{id}

**Описание:** Обновить соревнование

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request:**
```json
{
  "name": "Кубок России 2025 (обновлённое название)",
  "venue": "Новый адрес"
}
```

**Response:** `200 OK`

---

### DELETE /competitions/{id}

**Описание:** Удалить соревнование

**Headers:**
```
Authorization: Bearer {access_token}
```

**Response:** `204 No Content`

**Errors:**
- `403 Forbidden` — недостаточно прав
- `404 Not Found` — соревнование не найдено

---

## Athletes API

### GET /athletes

**Описание:** Получить список спортсменов

**Query Parameters:**
- `search` (string) — поиск по ФИО
- `country` (string) — фильтр по стране (ISO 3166-1)
- `club` (string) — фильтр по клубу
- `age_from` (int) — минимальный возраст
- `age_to` (int) — максимальный возраст

**Response:** `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "id": "athlete-uuid-1",
      "first_name": "Алина",
      "last_name": "Кабаева",
      "date_of_birth": "1983-05-12",
      "age": 42,
      "country": "RUS",
      "club": "Газпром",
      "region": "Москва",
      "photo_url": "https://cdn.example.com/athletes/kabayeva.jpg"
    }
  ],
  "meta": { ... }
}
```

---

### GET /athletes/{id}

**Описание:** Получить информацию о спортсмене

**Response:** `200 OK`
```json
{
  "success": true,
  "data": {
    "id": "athlete-uuid-1",
    "first_name": "Алина",
    "last_name": "Кабаева",
    "date_of_birth": "1983-05-12",
    "country": "RUS",
    "club": "Газпром",
    "region": "Москва",
    "competitions": [
      {
        "competition_id": "comp-uuid",
        "competition_name": "Первенство Москвы",
        "start_number": 12,
        "status": "confirmed"
      }
    ],
    "career_stats": {
      "total_competitions": 45,
      "gold_medals": 18,
      "best_score": 37.900
    }
  }
}
```

---

### POST /athletes

**Описание:** Добавить нового спортсмена

**Request:**
```json
{
  "first_name": "Дина",
  "last_name": "Аверина",
  "date_of_birth": "1998-08-13",
  "country": "RUS",
  "club": "Москва",
  "region": "Москва"
}
```

**Response:** `201 Created`

---

### POST /competitions/{comp_id}/athletes

**Описание:** Зарегистрировать спортсмена на соревнование

**Request:**
```json
{
  "athlete_id": "athlete-uuid-1",
  "start_number": 126
}
```

**Response:** `201 Created`
```json
{
  "success": true,
  "data": {
    "registration_id": "reg-uuid",
    "athlete_id": "athlete-uuid-1",
    "competition_id": "comp-uuid",
    "start_number": 126,
    "status": "registered",
    "registered_at": "2025-03-20T10:00:00Z"
  }
}
```

---

## Judges API

### GET /judges

**Описание:** Получить список судей

**Query Parameters:**
- `brevet` (string) — фильтр по категории: `1`, `2`, `3`, `4`, `5`
- `country` (string) — фильтр по стране
- `available` (boolean) — только доступные судьи

**Response:** `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "id": "judge-uuid-1",
      "first_name": "Мария",
      "last_name": "Иванова",
      "country": "RUS",
      "brevet_category": "4",
      "email": "ivanova@example.com",
      "is_active": true
    }
  ]
}
```

---

### POST /competitions/{comp_id}/judge-panels

**Описание:** Создать судейскую бригаду

**Request:**
```json
{
  "group_id": "group-uuid",
  "panel_type": "E",
  "judges": [
    {
      "judge_id": "judge-uuid-1",
      "position": 1
    },
    {
      "judge_id": "judge-uuid-2",
      "position": 2
    },
    {
      "judge_id": "judge-uuid-3",
      "position": 3
    },
    {
      "judge_id": "judge-uuid-4",
      "position": 4
    }
  ]
}
```

**Response:** `201 Created`

---

## Scores API

### POST /scores

**Описание:** Отправить оценку судьи

**Headers:**
```
Authorization: Bearer {access_token}
Content-Type: application/json
```

**Request (E-Panel Judge):**
```json
{
  "start_list_id": "startlist-uuid",
  "athlete_id": "athlete-uuid",
  "group_id": "group-uuid",
  "panel_id": "panel-uuid",
  "judge_id": "judge-uuid",
  "score_type": "E",
  "deductions": [
    {"type": "execution", "value": 0.10, "description": "Небольшая ошибка баланса"},
    {"type": "execution", "value": 0.30, "description": "Потеря предмета"},
    {"type": "execution", "value": 0.10, "description": "Неточность в элементе"}
  ],
  "total_deductions": 0.50,
  "score_value": 9.50
}
```

**Response:** `201 Created`
```json
{
  "success": true,
  "data": {
    "score_id": "score-uuid",
    "status": "submitted",
    "submitted_at": "2025-04-15T10:30:00Z"
  }
}
```

---

### GET /scores/start-list/{start_list_id}

**Описание:** Получить все оценки для выступления

**Response:** `200 OK`
```json
{
  "success": true,
  "data": {
    "start_list_id": "startlist-uuid",
    "athlete": {
      "id": "athlete-uuid",
      "first_name": "Алина",
      "last_name": "Кабаева",
      "start_number": 12
    },
    "scores": {
      "D": {
        "value": 9.400,
        "status": "confirmed",
        "breakdown": {
          "DB": 5.4,
          "DA": 2.4,
          "DS": 0.4,
          "DD": 1.2
        }
      },
      "E": {
        "value": 8.425,
        "status": "confirmed",
        "judge_scores": [
          {"judge_id": "j1", "deductions": 1.4},
          {"judge_id": "j2", "deductions": 1.5},
          {"judge_id": "j3", "deductions": 1.6},
          {"judge_id": "j4", "deductions": 1.8}
        ],
        "calculation": "Excluded: 1.4 (min), 2.0 (max). Average: 1.575"
      },
      "A": {
        "value": 8.550,
        "status": "confirmed"
      },
      "ND": {
        "value": -0.30,
        "deductions": [
          {"type": "out_of_bounds", "value": -0.30}
        ]
      },
      "final": {
        "value": 26.075,
        "formula": "9.400 + 8.425 + 8.550 - 0.30",
        "status": "confirmed",
        "rank": 1
      }
    }
  }
}
```

---

### POST /scores/calculate-final

**Описание:** Рассчитать итоговую оценку (вызывается секретарём)

**Request:**
```json
{
  "start_list_id": "startlist-uuid"
}
```

**Response:** `200 OK`
```json
{
  "success": true,
  "data": {
    "final_score": 26.075,
    "breakdown": {
      "d_score": 9.400,
      "e_score": 8.425,
      "a_score": 8.550,
      "neutral_deductions": -0.30,
      "penalties": 0.00
    },
    "rank": 1,
    "calculated_at": "2025-04-15T10:35:00Z"
  }
}
```

---

## Start Lists API

### GET /groups/{group_id}/start-list

**Описание:** Получить стартовый протокол группы

**Response:** `200 OK`
```json
{
  "success": true,
  "data": {
    "group_id": "group-uuid",
    "group_name": "Senior Individual All-Around",
    "scheduled_time": "10:00:00",
    "entries": [
      {
        "order_number": 1,
        "athlete": {
          "id": "athlete-uuid",
          "first_name": "Алина",
          "last_name": "Кабаева",
          "start_number": 12,
          "country": "RUS",
          "club": "Газпром"
        },
        "scheduled_time": "10:00:00",
        "status": "scheduled"
      },
      {
        "order_number": 2,
        "athlete": { ... },
        "scheduled_time": "10:03:30",
        "status": "scheduled"
      }
    ]
  }
}
```

---

### POST /groups/{group_id}/start-list

**Описание:** Сгенерировать стартовый протокол

**Request:**
```json
{
  "method": "random",
  "start_time": "10:00:00",
  "interval_minutes": 3.5
}
```

**Methods:**
- `random` — случайная жеребьёвка
- `by_rank` — по рейтингу (от худшего к лучшему)
- `manual` — ручной порядок

**Response:** `201 Created`

---

### PUT /start-lists/{id}/status

**Описание:** Обновить статус выступления

**Request:**
```json
{
  "status": "performing"
}
```

**Statuses:**
- `scheduled` — запланировано
- `warming_up` — разминка на ковре
- `performing` — выступление
- `completed` — завершено
- `dns` — не вышла на старт (Did Not Start)

**Response:** `200 OK`

---

## Events API

### GET /competitions/{comp_id}/events

**Описание:** Получить события соревнования

**Response:** `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "id": "event-uuid-1",
      "name": "Открытие соревнования",
      "event_type": "opening",
      "date": "2025-04-15",
      "start_time": "09:00:00",
      "duration_minutes": 30
    },
    {
      "id": "event-uuid-2",
      "name": "Опробование площадки",
      "event_type": "podium_training",
      "date": "2025-04-15",
      "start_time": "09:30:00",
      "duration_minutes": 60
    }
  ]
}
```

---

### POST /competitions/{comp_id}/events

**Описание:** Создать событие

**Request:**
```json
{
  "name": "Технический перерыв",
  "event_type": "break",
  "date_id": "date-uuid",
  "start_time": "12:00:00",
  "duration_minutes": 15
}
```

**Response:** `201 Created`

---

## WebSocket API

### Подключение

**URL:** `wss://api.rgsystem.example/v1/ws`

**Параметры:**
```
?token={access_token}&competition_id={comp_id}
```

---

### События (Events)

#### 1. Подключение

**Client → Server:**
```json
{
  "type": "subscribe",
  "channel": "competition",
  "competition_id": "comp-uuid"
}
```

**Server → Client:**
```json
{
  "type": "subscribed",
  "channel": "competition",
  "message": "Successfully subscribed"
}
```

---

#### 2. Новая оценка

**Server → Client:**
```json
{
  "type": "score_submitted",
  "data": {
    "start_list_id": "startlist-uuid",
    "athlete": {
      "first_name": "Алина",
      "last_name": "Кабаева",
      "start_number": 12
    },
    "panel_type": "E",
    "judge_position": 1,
    "submitted_at": "2025-04-15T10:30:00Z"
  }
}
```

---

#### 3. Итоговая оценка подтверждена

**Server → Client:**
```json
{
  "type": "score_confirmed",
  "data": {
    "start_list_id": "startlist-uuid",
    "athlete": {
      "first_name": "Алина",
      "last_name": "Кабаева",
      "start_number": 12
    },
    "final_score": 26.075,
    "rank": 1,
    "confirmed_at": "2025-04-15T10:35:00Z"
  }
}
```

---

#### 4. Обновление статуса выступления

**Server → Client:**
```json
{
  "type": "performance_status",
  "data": {
    "start_list_id": "startlist-uuid",
    "athlete": {
      "first_name": "Алина",
      "last_name": "Кабаева",
      "start_number": 12
    },
    "old_status": "warming_up",
    "new_status": "performing"
  }
}
```

---

## Коды ошибок

| HTTP Код | Код ошибки | Описание |
|----------|------------|----------|
| 400 | `BAD_REQUEST` | Некорректный запрос |
| 400 | `VALIDATION_ERROR` | Ошибка валидации данных |
| 401 | `UNAUTHORIZED` | Требуется аутентификация |
| 401 | `TOKEN_EXPIRED` | Токен истёк |
| 403 | `FORBIDDEN` | Недостаточно прав |
| 404 | `NOT_FOUND` | Ресурс не найден |
| 409 | `CONFLICT` | Конфликт данных (например, дубликат стартового номера) |
| 422 | `UNPROCESSABLE_ENTITY` | Невозможно обработать запрос |
| 429 | `RATE_LIMIT_EXCEEDED` | Превышен лимит запросов |
| 500 | `INTERNAL_SERVER_ERROR` | Внутренняя ошибка сервера |
| 503 | `SERVICE_UNAVAILABLE` | Сервис временно недоступен |

---

## Примеры использования

### Пример 1: Создание соревнования и регистрация спортсменов

```bash
# 1. Войти в систему
curl -X POST https://api.rgsystem.example/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@example.com",
    "password": "password123"
  }'

# Response: {"data": {"access_token": "TOKEN", ...}}

# 2. Создать соревнование
curl -X POST https://api.rgsystem.example/v1/competitions \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Кубок Москвы",
    "city": "Москва",
    "venue": "СК Олимпийский",
    "date_start": "2025-06-01",
    "date_end": "2025-06-03"
  }'

# Response: {"data": {"id": "COMP_ID", ...}}

# 3. Зарегистрировать спортсменку
curl -X POST https://api.rgsystem.example/v1/competitions/COMP_ID/athletes \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "athlete_id": "ATHLETE_ID",
    "start_number": 1
  }'
```

---

### Пример 2: Судья выставляет оценку

```javascript
// JavaScript (Frontend)
const submitScore = async () => {
  const response = await fetch('https://api.rgsystem.example/v1/scores', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      start_list_id: 'startlist-uuid',
      athlete_id: 'athlete-uuid',
      group_id: 'group-uuid',
      panel_id: 'panel-uuid',
      judge_id: 'judge-uuid',
      score_type: 'E',
      deductions: [
        { type: 'execution', value: 0.10, description: 'Ошибка баланса' },
        { type: 'execution', value: 0.30, description: 'Потеря предмета' }
      ],
      total_deductions: 0.40,
      score_value: 9.60
    })
  });

  const data = await response.json();
  console.log('Score submitted:', data);
};
```

---

### Пример 3: WebSocket для реального времени

```javascript
// JavaScript WebSocket Client
const ws = new WebSocket('wss://api.rgsystem.example/v1/ws?token=TOKEN&competition_id=COMP_ID');

ws.onopen = () => {
  console.log('Connected to WebSocket');

  // Подписаться на канал соревнования
  ws.send(JSON.stringify({
    type: 'subscribe',
    channel: 'competition',
    competition_id: 'COMP_ID'
  }));
};

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);

  switch (message.type) {
    case 'score_confirmed':
      console.log('New score:', message.data);
      updateResultsTable(message.data);
      break;

    case 'performance_status':
      console.log('Status updated:', message.data);
      updateAthleteStatus(message.data);
      break;
  }
};
```

---

## Rate Limiting

**Лимиты запросов:**
- Аутентифицированные пользователи: **1000 запросов/час**
- Неаутентифицированные: **100 запросов/час**

**Headers:**
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 987
X-RateLimit-Reset: 1616155200
```

---

## Версионирование API

**Поддерживаемые версии:**
- `v1` (текущая)

**Устаревшие версии:**
- Нет

**Deprecation policy:** За 6 месяцев до удаления версии

---

**Конец документа**

> **Документация OpenAPI/Swagger:** https://api.rgsystem.example/v1/docs
> **Postman Collection:** https://www.postman.com/rgsystem/collection
