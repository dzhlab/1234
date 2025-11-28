# Примеры API запросов и ответов
# API Request/Response Examples

> **Версия API:** 1.0
> **Base URL:** `https://api.rgsystem.local`
> **Формат:** JSON
> **Кодировка:** UTF-8

---

## Содержание / Table of Contents

1. [Authentication API](#authentication-api)
2. [Competitions API](#competitions-api)
3. [Athletes API](#athletes-api)
4. [Judges API](#judges-api)
5. [Scores API](#scores-api)
6. [Start Lists API](#start-lists-api)
7. [Events API](#events-api)
8. [WebSocket Events](#websocket-events)
9. [Error Responses](#error-responses)

---

## Authentication API

### POST /api/auth/login

Аутентификация пользователя и получение JWT токена.

**Request:**
```json
{
  "email": "admin@rgsystem.local",
  "password": "SecurePassword123!"
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiMTIzZTQ1NjctZTg5Yi0xMmQzLWE0NTYtNDI2NjE0MTc0MDAwIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNjQwOTk1MjAwLCJleHAiOjE2NDA5OTg4MDB9.abc123def456",
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiMTIzZTQ1NjctZTg5Yi0xMmQzLWE0NTYtNDI2NjE0MTc0MDAwIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNjQwOTk1MjAwLCJleHAiOjE2NDE2MDAwMDB9.xyz789uvw456",
    "token_type": "Bearer",
    "expires_in": 3600,
    "user": {
      "id": "123e4567-e89b-12d3-a456-426614174000",
      "email": "admin@rgsystem.local",
      "first_name": "Иван",
      "last_name": "Иванов",
      "role": "admin",
      "permissions": [
        "competitions.create",
        "competitions.update",
        "competitions.delete",
        "users.manage",
        "system.settings"
      ],
      "created_at": "2025-01-01T10:00:00.000Z",
      "last_login": "2025-01-15T14:30:00.000Z"
    }
  }
}
```

**Error Response: 401 Unauthorized**
```json
{
  "status": 401,
  "code": "RG-AUTH-PERM-001",
  "message": "Invalid email or password",
  "path": "/api/auth/login",
  "timestamp": "2025-01-15T14:30:00.123Z",
  "trace_id": "a7b3c4d5-e6f7-8901-2345-6789abcdef01"
}
```

---

### POST /api/auth/refresh

Обновление access токена с помощью refresh токена.

**Request:**
```json
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.new_token...",
    "expires_in": 3600
  }
}
```

---

### POST /api/auth/logout

Выход из системы (аннулирование токенов).

**Request Headers:**
```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response: 204 No Content**

---

## Competitions API

### GET /api/competitions

Получение списка соревнований с пагинацией и фильтрацией.

**Request:**
```
GET /api/competitions?page=1&limit=10&status=active&city=Moscow&sort=date_start:desc
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "competitions": [
      {
        "id": "456e4567-e89b-12d3-a456-426614174002",
        "name": "Кубок Москвы по художественной гимнастике 2025",
        "city": "Москва",
        "venue": "СК «Олимпийский»",
        "date_start": "2025-03-15",
        "date_end": "2025-03-17",
        "status": "active",
        "level": "national",
        "age_categories": ["junior", "youth", "senior"],
        "apparatus": ["rope", "hoop", "ball", "clubs", "ribbon"],
        "organizer": {
          "id": "789e4567-e89b-12d3-a456-426614174001",
          "name": "Московская федерация гимнастики",
          "email": "info@moscowgymnastics.ru"
        },
        "participants_count": 156,
        "judges_count": 24,
        "settings": {
          "allow_public_viewing": true,
          "enable_live_scoring": true,
          "tiebreak_method": "e_score_comparison",
          "max_athletes_per_club": 10
        },
        "created_at": "2025-01-10T09:00:00.000Z",
        "updated_at": "2025-01-14T15:30:00.000Z"
      },
      {
        "id": "567e4567-e89b-12d3-a456-426614174003",
        "name": "Первенство России среди юниоров 2025",
        "city": "Санкт-Петербург",
        "venue": "Дворец спорта «Юбилейный»",
        "date_start": "2025-04-20",
        "date_end": "2025-04-23",
        "status": "draft",
        "level": "national",
        "age_categories": ["junior"],
        "apparatus": ["rope", "ball", "clubs", "ribbon"],
        "organizer": {
          "id": "890e4567-e89b-12d3-a456-426614174004",
          "name": "Всероссийская федерация художественной гимнастики",
          "email": "info@vfrg.ru"
        },
        "participants_count": 0,
        "judges_count": 0,
        "settings": {
          "allow_public_viewing": false,
          "enable_live_scoring": true,
          "tiebreak_method": "e_score_comparison",
          "max_athletes_per_club": 5
        },
        "created_at": "2025-01-12T11:00:00.000Z",
        "updated_at": "2025-01-12T11:00:00.000Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 10,
      "total": 45,
      "total_pages": 5,
      "has_next": true,
      "has_prev": false
    }
  }
}
```

---

### GET /api/competitions/:id

Получение детальной информации о соревновании.

**Request:**
```
GET /api/competitions/456e4567-e89b-12d3-a456-426614174002
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": "456e4567-e89b-12d3-a456-426614174002",
    "name": "Кубок Москвы по художественной гимнастике 2025",
    "description": "Традиционное ежегодное соревнование по художественной гимнастике среди спортсменок из регионов России",
    "city": "Москва",
    "venue": "СК «Олимпийский»",
    "address": "Олимпийский проспект, 16, Москва, 129110",
    "date_start": "2025-03-15",
    "date_end": "2025-03-17",
    "status": "active",
    "level": "national",
    "format": "individual_all_around",
    "age_categories": [
      {
        "code": "junior",
        "name": "Юниоры",
        "age_min": 13,
        "age_max": 15
      },
      {
        "code": "youth",
        "name": "Юноши",
        "age_min": 16,
        "age_max": 18
      },
      {
        "code": "senior",
        "name": "Взрослые",
        "age_min": 19,
        "age_max": null
      }
    ],
    "apparatus": [
      { "code": "rope", "name": "Скакалка" },
      { "code": "hoop", "name": "Обруч" },
      { "code": "ball", "name": "Мяч" },
      { "code": "clubs", "name": "Булавы" },
      { "code": "ribbon", "name": "Лента" }
    ],
    "organizer": {
      "id": "789e4567-e89b-12d3-a456-426614174001",
      "name": "Московская федерация гимнастики",
      "email": "info@moscowgymnastics.ru",
      "phone": "+7 (495) 123-45-67",
      "website": "https://moscowgymnastics.ru"
    },
    "chief_judge": {
      "id": "234e4567-e89b-12d3-a456-426614174005",
      "first_name": "Елена",
      "last_name": "Петрова",
      "brevet": 3,
      "email": "e.petrova@judges.ru"
    },
    "secretary": {
      "id": "345e4567-e89b-12d3-a456-426614174006",
      "first_name": "Ольга",
      "last_name": "Сидорова",
      "email": "o.sidorova@secretary.ru"
    },
    "groups": [
      {
        "id": "678e4567-e89b-12d3-a456-426614174007",
        "name": "Группа 1 - Юниоры (Скакалка)",
        "age_category": "junior",
        "apparatus": "rope",
        "date": "2025-03-15",
        "time_start": "10:00",
        "athletes_count": 18
      },
      {
        "id": "789e4567-e89b-12d3-a456-426614174008",
        "name": "Группа 2 - Юниоры (Обруч)",
        "age_category": "junior",
        "apparatus": "hoop",
        "date": "2025-03-15",
        "time_start": "14:00",
        "athletes_count": 18
      }
    ],
    "participants_count": 156,
    "judges_count": 24,
    "settings": {
      "allow_public_viewing": true,
      "enable_live_scoring": true,
      "enable_video_review": true,
      "tiebreak_method": "e_score_comparison",
      "max_athletes_per_club": 10,
      "warm_up_duration": 30,
      "performance_time_limit": 90,
      "judging_system": "FIG_2025_2028",
      "penalty_out_of_bounds": 0.3,
      "penalty_time_violation": 0.5
    },
    "created_at": "2025-01-10T09:00:00.000Z",
    "updated_at": "2025-01-14T15:30:00.000Z"
  }
}
```

---

### POST /api/competitions

Создание нового соревнования.

**Request:**
```json
{
  "name": "Открытый кубок Сибири 2025",
  "description": "Региональное соревнование по художественной гимнастике",
  "city": "Новосибирск",
  "venue": "ДС «Сибирь»",
  "address": "ул. Ленина, 35, Новосибирск, 630099",
  "date_start": "2025-05-10",
  "date_end": "2025-05-12",
  "level": "regional",
  "format": "individual_all_around",
  "age_categories": ["junior", "youth"],
  "apparatus": ["rope", "hoop", "ball", "clubs"],
  "settings": {
    "allow_public_viewing": true,
    "enable_live_scoring": true,
    "enable_video_review": false,
    "tiebreak_method": "e_score_comparison",
    "max_athletes_per_club": 8
  }
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "id": "901e4567-e89b-12d3-a456-426614174009",
    "name": "Открытый кубок Сибири 2025",
    "description": "Региональное соревнование по художественной гимнастике",
    "city": "Новосибирск",
    "venue": "ДС «Сибирь»",
    "address": "ул. Ленина, 35, Новосибирск, 630099",
    "date_start": "2025-05-10",
    "date_end": "2025-05-12",
    "status": "draft",
    "level": "regional",
    "format": "individual_all_around",
    "age_categories": ["junior", "youth"],
    "apparatus": ["rope", "hoop", "ball", "clubs"],
    "organizer": {
      "id": "789e4567-e89b-12d3-a456-426614174001",
      "name": "Иван Иванов",
      "email": "admin@rgsystem.local"
    },
    "participants_count": 0,
    "judges_count": 0,
    "settings": {
      "allow_public_viewing": true,
      "enable_live_scoring": true,
      "enable_video_review": false,
      "tiebreak_method": "e_score_comparison",
      "max_athletes_per_club": 8,
      "warm_up_duration": 30,
      "performance_time_limit": 90,
      "judging_system": "FIG_2025_2028"
    },
    "created_at": "2025-01-15T10:00:00.000Z",
    "updated_at": "2025-01-15T10:00:00.000Z"
  }
}
```

---

### PUT /api/competitions/:id

Обновление информации о соревновании.

**Request:**
```json
{
  "name": "Открытый кубок Сибири 2025 (обновлено)",
  "date_start": "2025-05-15",
  "date_end": "2025-05-17",
  "settings": {
    "max_athletes_per_club": 10
  }
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "id": "901e4567-e89b-12d3-a456-426614174009",
    "name": "Открытый кубок Сибири 2025 (обновлено)",
    "date_start": "2025-05-15",
    "date_end": "2025-05-17",
    "settings": {
      "max_athletes_per_club": 10
    },
    "updated_at": "2025-01-15T11:00:00.000Z"
  }
}
```

---

### DELETE /api/competitions/:id

Удаление соревнования (только в статусе draft).

**Request:**
```
DELETE /api/competitions/901e4567-e89b-12d3-a456-426614174009
```

**Response: 204 No Content**

**Error Response: 409 Conflict**
```json
{
  "status": 409,
  "code": "RG-COMP-STATE-001",
  "message": "Cannot delete competition in 'active' status. Only draft competitions can be deleted.",
  "path": "/api/competitions/901e4567-e89b-12d3-a456-426614174009",
  "timestamp": "2025-01-15T11:30:00.123Z",
  "trace_id": "b8c4d5e6-f7a8-9012-3456-789abcdef012"
}
```

---

## Athletes API

### GET /api/athletes

Получение списка спортсменов.

**Request:**
```
GET /api/athletes?page=1&limit=20&search=Иванова&club=Динамо&age_category=junior
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "athletes": [
      {
        "id": "111e4567-e89b-12d3-a456-426614174010",
        "first_name": "Анна",
        "last_name": "Иванова",
        "middle_name": "Сергеевна",
        "date_of_birth": "2008-05-20",
        "age": 16,
        "gender": "female",
        "club": {
          "id": "222e4567-e89b-12d3-a456-426614174011",
          "name": "СДЮШОР Динамо",
          "city": "Москва",
          "region": "Московская область"
        },
        "coach": {
          "id": "333e4567-e89b-12d3-a456-426614174012",
          "first_name": "Мария",
          "last_name": "Петрова",
          "phone": "+7 (916) 123-45-67"
        },
        "rank": "КМС",
        "photo_url": "https://cdn.rgsystem.local/athletes/111e4567.jpg",
        "competitions_count": 12,
        "created_at": "2024-01-15T10:00:00.000Z",
        "updated_at": "2025-01-10T14:30:00.000Z"
      }
    ],
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 1,
      "total_pages": 1,
      "has_next": false,
      "has_prev": false
    }
  }
}
```

---

### POST /api/competitions/:id/athletes

Регистрация спортсмена на соревнование.

**Request:**
```json
{
  "athlete_id": "111e4567-e89b-12d3-a456-426614174010",
  "age_category": "junior",
  "apparatus": ["rope", "hoop", "ball", "clubs", "ribbon"],
  "is_individual": true,
  "bib_number": 45
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "registration_id": "444e4567-e89b-12d3-a456-426614174013",
    "competition_id": "456e4567-e89b-12d3-a456-426614174002",
    "athlete": {
      "id": "111e4567-e89b-12d3-a456-426614174010",
      "first_name": "Анна",
      "last_name": "Иванова",
      "date_of_birth": "2008-05-20",
      "club": "СДЮШОР Динамо"
    },
    "age_category": "junior",
    "apparatus": ["rope", "hoop", "ball", "clubs", "ribbon"],
    "is_individual": true,
    "bib_number": 45,
    "status": "registered",
    "registered_at": "2025-01-15T12:00:00.000Z"
  }
}
```

---

## Judges API

### POST /api/competitions/:id/judge-panels

Создание судейской бригады для группы.

**Request:**
```json
{
  "group_id": "678e4567-e89b-12d3-a456-426614174007",
  "panel_type": "D",
  "judges": [
    {
      "judge_id": "555e4567-e89b-12d3-a456-426614174014",
      "position": 1
    },
    {
      "judge_id": "666e4567-e89b-12d3-a456-426614174015",
      "position": 2
    },
    {
      "judge_id": "777e4567-e89b-12d3-a456-426614174016",
      "position": 3
    },
    {
      "judge_id": "888e4567-e89b-12d3-a456-426614174017",
      "position": 4
    }
  ]
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "panel_id": "999e4567-e89b-12d3-a456-426614174018",
    "group_id": "678e4567-e89b-12d3-a456-426614174007",
    "panel_type": "D",
    "judges": [
      {
        "judge_id": "555e4567-e89b-12d3-a456-426614174014",
        "first_name": "Елена",
        "last_name": "Соколова",
        "brevet": 3,
        "position": 1
      },
      {
        "judge_id": "666e4567-e89b-12d3-a456-426614174015",
        "first_name": "Ирина",
        "last_name": "Кузнецова",
        "brevet": 3,
        "position": 2
      },
      {
        "judge_id": "777e4567-e89b-12d3-a456-426614174016",
        "first_name": "Наталья",
        "last_name": "Смирнова",
        "brevet": 4,
        "position": 3
      },
      {
        "judge_id": "888e4567-e89b-12d3-a456-426614174017",
        "first_name": "Светлана",
        "last_name": "Морозова",
        "brevet": 3,
        "position": 4
      }
    ],
    "created_at": "2025-01-15T13:00:00.000Z"
  }
}
```

---

## Scores API

### POST /api/scores

Отправка оценки судьей.

**Request (D-Panel):**
```json
{
  "performance_id": "aaa-4567-e89b-12d3-a456-426614174019",
  "panel_type": "D",
  "judge_id": "555e4567-e89b-12d3-a456-426614174014",
  "score_components": {
    "DB": 3.5,
    "DA": 2.4,
    "DS": 0.4,
    "DD": 1.8
  },
  "total": 8.1,
  "notes": "Отличная техника, высокая сложность"
}
```

**Request (E-Panel):**
```json
{
  "performance_id": "aaa-4567-e89b-12d3-a456-426614174019",
  "panel_type": "E",
  "judge_id": "bbb-4567-e89b-12d3-a456-426614174020",
  "deductions": [
    {
      "type": "technical",
      "code": "E1",
      "description": "Неточность в повороте",
      "value": 0.1
    },
    {
      "type": "technical",
      "code": "E2",
      "description": "Небольшая потеря равновесия",
      "value": 0.2
    }
  ],
  "total_deductions": 0.3,
  "final_score": 9.7,
  "notes": ""
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "score_id": "ccc-4567-e89b-12d3-a456-426614174021",
    "performance_id": "aaa-4567-e89b-12d3-a456-426614174019",
    "panel_type": "D",
    "judge_id": "555e4567-e89b-12d3-a456-426614174014",
    "judge_name": "Елена Соколова",
    "score_components": {
      "DB": 3.5,
      "DA": 2.4,
      "DS": 0.4,
      "DD": 1.8
    },
    "total": 8.1,
    "status": "submitted",
    "notes": "Отличная техника, высокая сложность",
    "submitted_at": "2025-03-15T10:25:30.000Z"
  }
}
```

---

### POST /api/scores/calculate-final

Расчет финальной оценки для выступления.

**Request:**
```json
{
  "performance_id": "aaa-4567-e89b-12d3-a456-426614174019"
}
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "performance_id": "aaa-4567-e89b-12d3-a456-426614174019",
    "athlete": {
      "id": "111e4567-e89b-12d3-a456-426614174010",
      "first_name": "Анна",
      "last_name": "Иванова",
      "bib_number": 45,
      "club": "СДЮШОР Динамо"
    },
    "apparatus": "rope",
    "d_score": {
      "judges": [
        { "judge_id": "555e4567", "name": "Е. Соколова", "score": 8.1 },
        { "judge_id": "666e4567", "name": "И. Кузнецова", "score": 8.2 },
        { "judge_id": "777e4567", "name": "Н. Смирнова", "score": 7.9 },
        { "judge_id": "888e4567", "name": "С. Морозова", "score": 8.0 }
      ],
      "average": 8.05,
      "final": 8.05
    },
    "e_score": {
      "judges": [
        { "judge_id": "aaa", "name": "Т. Волкова", "deductions": 0.3, "score": 9.7 },
        { "judge_id": "bbb", "name": "О. Лебедева", "deductions": 0.4, "score": 9.6 },
        { "judge_id": "ccc", "name": "М. Зайцева", "deductions": 0.2, "score": 9.8 },
        { "judge_id": "ddd", "name": "А. Орлова", "deductions": 0.5, "score": 9.5 },
        { "judge_id": "eee", "name": "Л. Соловьева", "deductions": 0.3, "score": 9.7 },
        { "judge_id": "fff", "name": "Н. Медведева", "deductions": 0.4, "score": 9.6 }
      ],
      "excluded": [
        { "judge_id": "ddd", "score": 9.5, "reason": "min" },
        { "judge_id": "ccc", "score": 9.8, "reason": "max" }
      ],
      "average": 9.65,
      "final": 9.65
    },
    "a_score": null,
    "neutral_deductions": [
      {
        "type": "time_violation",
        "description": "Превышение времени на 2 секунды",
        "value": 0.05
      }
    ],
    "total_nd": 0.05,
    "penalties": [],
    "total_penalties": 0.0,
    "calculation": {
      "d_score": 8.05,
      "e_score": 9.65,
      "a_score": 0.0,
      "neutral_deductions": 0.05,
      "penalties": 0.0
    },
    "final_score": 17.65,
    "formula": "Final Score = D (8.05) + E (9.65) + A (0.0) - ND (0.05) - Penalties (0.0) = 17.65",
    "calculated_at": "2025-03-15T10:30:00.000Z",
    "calculated_by": "system"
  }
}
```

---

## Start Lists API

### GET /api/groups/:id/start-list

Получение стартового протокола для группы.

**Request:**
```
GET /api/groups/678e4567-e89b-12d3-a456-426614174007/start-list
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "group_id": "678e4567-e89b-12d3-a456-426614174007",
    "group_name": "Группа 1 - Юниоры (Скакалка)",
    "age_category": "junior",
    "apparatus": "rope",
    "date": "2025-03-15",
    "time_start": "10:00",
    "athletes": [
      {
        "start_order": 1,
        "bib_number": 12,
        "athlete": {
          "id": "111e4567",
          "first_name": "Анна",
          "last_name": "Иванова",
          "club": "СДЮШОР Динамо",
          "city": "Москва"
        },
        "estimated_time": "10:00",
        "status": "registered"
      },
      {
        "start_order": 2,
        "bib_number": 25,
        "athlete": {
          "id": "222e4567",
          "first_name": "Мария",
          "last_name": "Петрова",
          "club": "Грация",
          "city": "Санкт-Петербург"
        },
        "estimated_time": "10:02",
        "status": "registered"
      },
      {
        "start_order": 3,
        "bib_number": 33,
        "athlete": {
          "id": "333e4567",
          "first_name": "Екатерина",
          "last_name": "Сидорова",
          "club": "Олимп",
          "city": "Казань"
        },
        "estimated_time": "10:04",
        "status": "registered"
      }
    ],
    "total_athletes": 18,
    "created_at": "2025-03-10T15:00:00.000Z",
    "updated_at": "2025-03-14T18:30:00.000Z"
  }
}
```

---

### POST /api/groups/:id/start-list

Создание/генерация стартового протокола.

**Request:**
```json
{
  "method": "random",
  "seed": 12345
}
```

**Другие методы:**
```json
{
  "method": "manual",
  "athletes": [
    { "athlete_id": "111e4567", "start_order": 1 },
    { "athlete_id": "222e4567", "start_order": 2 },
    { "athlete_id": "333e4567", "start_order": 3 }
  ]
}
```

**Response: 201 Created**
```json
{
  "success": true,
  "data": {
    "group_id": "678e4567-e89b-12d3-a456-426614174007",
    "method": "random",
    "athletes": [
      { "start_order": 1, "bib_number": 12, "athlete_id": "111e4567" },
      { "start_order": 2, "bib_number": 25, "athlete_id": "222e4567" },
      { "start_order": 3, "bib_number": 33, "athlete_id": "333e4567" }
    ],
    "total_athletes": 18,
    "created_at": "2025-03-10T15:00:00.000Z"
  }
}
```

---

### PUT /api/start-lists/:id/status

Обновление статуса участника в стартовом протоколе.

**Request:**
```json
{
  "status": "performing"
}
```

**Возможные статусы:**
- `registered` - зарегистрирован
- `warming_up` - разминка
- `ready` - готов к выступлению
- `performing` - выступает
- `judging` - оценивается
- `completed` - завершено
- `withdrawn` - снят

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "start_list_id": "ddd-4567-e89b-12d3-a456-426614174022",
    "athlete_id": "111e4567",
    "status": "performing",
    "updated_at": "2025-03-15T10:25:00.000Z"
  }
}
```

---

## Events API

### GET /api/competitions/:id/events

Получение событий соревнования (для аудита и мониторинга).

**Request:**
```
GET /api/competitions/456e4567/events?limit=50&type=score_submitted&from=2025-03-15T10:00:00Z
```

**Response: 200 OK**
```json
{
  "success": true,
  "data": {
    "events": [
      {
        "id": "eee-4567-e89b-12d3-a456-426614174023",
        "type": "score_submitted",
        "competition_id": "456e4567",
        "group_id": "678e4567",
        "performance_id": "aaa-4567",
        "actor": {
          "id": "555e4567",
          "name": "Елена Соколова",
          "role": "judge"
        },
        "data": {
          "panel_type": "D",
          "score": 8.1,
          "athlete": "Анна Иванова",
          "apparatus": "rope"
        },
        "timestamp": "2025-03-15T10:25:30.000Z"
      },
      {
        "id": "fff-4567-e89b-12d3-a456-426614174024",
        "type": "performance_started",
        "competition_id": "456e4567",
        "group_id": "678e4567",
        "performance_id": "aaa-4567",
        "actor": {
          "id": "system",
          "name": "System",
          "role": "system"
        },
        "data": {
          "athlete": "Анна Иванова",
          "bib_number": 45,
          "apparatus": "rope",
          "start_order": 1
        },
        "timestamp": "2025-03-15T10:23:00.000Z"
      }
    ],
    "pagination": {
      "limit": 50,
      "total": 245,
      "has_more": true
    }
  }
}
```

---

## WebSocket Events

### Connection

```javascript
const socket = io('wss://api.rgsystem.local/ws', {
  auth: {
    token: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'
  }
});

socket.on('connect', () => {
  console.log('Connected to WebSocket');
});
```

### Subscribe to Competition

**Client → Server:**
```json
{
  "event": "subscribe_competition",
  "data": {
    "competition_id": "456e4567-e89b-12d3-a456-426614174002"
  }
}
```

**Server → Client (confirmation):**
```json
{
  "event": "subscribed",
  "data": {
    "competition_id": "456e4567-e89b-12d3-a456-426614174002",
    "groups": ["678e4567", "789e4567"]
  }
}
```

### Score Submitted Event

**Server → Client:**
```json
{
  "event": "score_submitted",
  "data": {
    "competition_id": "456e4567",
    "group_id": "678e4567",
    "performance_id": "aaa-4567",
    "panel_type": "D",
    "judge": {
      "id": "555e4567",
      "name": "Елена Соколова"
    },
    "score": 8.1,
    "athlete": {
      "id": "111e4567",
      "name": "Анна Иванова",
      "bib_number": 45
    },
    "apparatus": "rope",
    "timestamp": "2025-03-15T10:25:30.000Z"
  }
}
```

### Final Score Calculated Event

**Server → Client:**
```json
{
  "event": "final_score_calculated",
  "data": {
    "competition_id": "456e4567",
    "group_id": "678e4567",
    "performance_id": "aaa-4567",
    "athlete": {
      "id": "111e4567",
      "name": "Анна Иванова",
      "bib_number": 45,
      "club": "СДЮШОР Динамо"
    },
    "apparatus": "rope",
    "scores": {
      "d_score": 8.05,
      "e_score": 9.65,
      "a_score": 0.0,
      "neutral_deductions": 0.05,
      "penalties": 0.0,
      "final_score": 17.65
    },
    "rank": 3,
    "timestamp": "2025-03-15T10:30:00.000Z"
  }
}
```

### Performance Status Changed Event

**Server → Client:**
```json
{
  "event": "performance_status_changed",
  "data": {
    "competition_id": "456e4567",
    "group_id": "678e4567",
    "performance_id": "aaa-4567",
    "athlete": {
      "id": "111e4567",
      "name": "Анна Иванова",
      "bib_number": 45
    },
    "old_status": "warming_up",
    "new_status": "performing",
    "timestamp": "2025-03-15T10:23:00.000Z"
  }
}
```

---

## Error Responses

### 400 Bad Request - Validation Error

```json
{
  "status": 400,
  "code": "RG-API-VAL-001",
  "message": "Validation failed",
  "details": [
    {
      "field": "email",
      "code": "invalid_format",
      "message": "Email must be a valid email address",
      "value": "invalid-email"
    },
    {
      "field": "password",
      "code": "too_short",
      "message": "Password must be at least 8 characters",
      "value": "***"
    }
  ],
  "path": "/api/auth/login",
  "timestamp": "2025-01-15T14:30:00.123Z",
  "trace_id": "a7b3c4d5-e6f7-8901-2345-6789abcdef01"
}
```

### 401 Unauthorized

```json
{
  "status": 401,
  "code": "RG-AUTH-STATE-001",
  "message": "Your session has expired. Please log in again.",
  "path": "/api/competitions",
  "timestamp": "2025-01-15T14:30:00.123Z",
  "trace_id": "b8c4d5e6-f7a8-9012-3456-789abcdef012"
}
```

### 403 Forbidden

```json
{
  "status": 403,
  "code": "RG-COMP-PERM-001",
  "message": "You don't have permission to modify this competition. Only the organizer can make changes.",
  "path": "/api/competitions/456e4567",
  "timestamp": "2025-01-15T14:30:00.123Z",
  "trace_id": "c9d5e6f7-a8b9-0123-4567-89abcdef0123"
}
```

### 404 Not Found

```json
{
  "status": 404,
  "code": "RG-API-VAL-002",
  "message": "Competition with ID '456e4567' not found",
  "path": "/api/competitions/456e4567",
  "timestamp": "2025-01-15T14:30:00.123Z",
  "trace_id": "d0e6f7a8-b9c0-1234-5678-9abcdef01234"
}
```

### 409 Conflict

```json
{
  "status": 409,
  "code": "RG-JUDGE-STATE-001",
  "message": "Score has already been confirmed and cannot be modified. Please contact the Chief Judge to unlock.",
  "path": "/api/scores/ccc-4567",
  "timestamp": "2025-01-15T14:30:00.123Z",
  "trace_id": "e1f7a8b9-c0d1-2345-6789-abcdef012345"
}
```

### 429 Too Many Requests

```json
{
  "status": 429,
  "code": "RG-API-PERM-001",
  "message": "Too many requests. Please try again later.",
  "retry_after": 3600,
  "path": "/api/competitions",
  "timestamp": "2025-01-15T14:30:00.123Z",
  "trace_id": "f2a8b9c0-d1e2-3456-789a-bcdef0123456"
}
```

### 500 Internal Server Error

```json
{
  "status": 500,
  "code": "RG-API-SYS-001",
  "message": "An unexpected error occurred. Our team has been notified and is working to fix it.",
  "path": "/api/scores/calculate",
  "timestamp": "2025-01-15T14:30:00.123Z",
  "trace_id": "a3b9c0d1-e2f3-4567-89ab-cdef01234567"
}
```

---

## Примеры использования / Usage Examples

### cURL

**Авторизация:**
```bash
curl -X POST https://api.rgsystem.local/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@rgsystem.local",
    "password": "SecurePassword123!"
  }'
```

**Получение списка соревнований:**
```bash
curl -X GET "https://api.rgsystem.local/api/competitions?page=1&limit=10" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Создание соревнования:**
```bash
curl -X POST https://api.rgsystem.local/api/competitions \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d @competition.json
```

### JavaScript (fetch)

```javascript
// Авторизация
const login = async (email, password) => {
  const response = await fetch('https://api.rgsystem.local/api/auth/login', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({ email, password })
  });

  const data = await response.json();

  if (data.success) {
    localStorage.setItem('access_token', data.data.access_token);
    localStorage.setItem('refresh_token', data.data.refresh_token);
    return data.data.user;
  } else {
    throw new Error(data.message);
  }
};

// Получение соревнований
const getCompetitions = async (page = 1, limit = 10) => {
  const token = localStorage.getItem('access_token');

  const response = await fetch(
    `https://api.rgsystem.local/api/competitions?page=${page}&limit=${limit}`,
    {
      headers: {
        'Authorization': `Bearer ${token}`
      }
    }
  );

  const data = await response.json();
  return data.data.competitions;
};

// Отправка оценки
const submitScore = async (performanceId, panelType, scoreData) => {
  const token = localStorage.getItem('access_token');

  const response = await fetch('https://api.rgsystem.local/api/scores', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      performance_id: performanceId,
      panel_type: panelType,
      ...scoreData
    })
  });

  const data = await response.json();
  return data.data;
};
```

### Python (requests)

```python
import requests

BASE_URL = "https://api.rgsystem.local/api"

# Авторизация
def login(email, password):
    response = requests.post(
        f"{BASE_URL}/auth/login",
        json={"email": email, "password": password}
    )
    data = response.json()

    if data["success"]:
        return data["data"]["access_token"]
    else:
        raise Exception(data["message"])

# Получение соревнований
def get_competitions(token, page=1, limit=10):
    response = requests.get(
        f"{BASE_URL}/competitions",
        headers={"Authorization": f"Bearer {token}"},
        params={"page": page, "limit": limit}
    )
    data = response.json()
    return data["data"]["competitions"]

# Создание соревнования
def create_competition(token, competition_data):
    response = requests.post(
        f"{BASE_URL}/competitions",
        headers={
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json"
        },
        json=competition_data
    )
    data = response.json()
    return data["data"]

# Использование
token = login("admin@rgsystem.local", "SecurePassword123!")
competitions = get_competitions(token)
print(f"Found {len(competitions)} competitions")
```

---

## Postman Collection

Для удобства тестирования API вы можете импортировать [Postman коллекцию](./postman-collection.json) со всеми примерами запросов.

---

**Последнее обновление:** 2025-11-27
**Версия API:** 1.0
**Документация:** См. [API_SPECIFICATION.md](./API_SPECIFICATION.md)
