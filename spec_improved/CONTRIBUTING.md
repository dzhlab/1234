# Руководство для разработчиков
# Contributing Guide

> Добро пожаловать в проект RG System! Мы рады вашему желанию внести вклад.

---

## Содержание

1. [С чего начать](#с-чего-начать)
2. [Настройка окружения](#настройка-окружения)
3. [Процесс разработки](#процесс-разработки)
4. [Code Style и стандарты](#code-style-и-стандарты)
5. [Тестирование](#тестирование)
6. [Pull Request процесс](#pull-request-процесс)
7. [Архитектурные решения](#архитектурные-решения)
8. [База данных](#база-данных)
9. [API разработка](#api-разработка)
10. [Frontend разработка](#frontend-разработка)
11. [Документация](#документация)
12. [Получение помощи](#получение-помощи)

---

## С чего начать

### Для новых контрибьюторов

1. **Изучите документацию:**
   - [README.md](./README.md) — навигация по документации
   - [ANALYSIS_REPORT.md](./ANALYSIS_REPORT.md) — анализ системы
   - [DIAGRAMS.md](./DIAGRAMS.md) — архитектура
   - [DATABASE_SCHEMA.md](./DATABASE_SCHEMA.md) — схема БД
   - [API_SPECIFICATION.md](./API_SPECIFICATION.md) — API

2. **Найдите задачу:**
   - Посмотрите issue с метками `good-first-issue` или `help-wanted`
   - Изучите [TODO.md](./TODO.md) для идей

3. **Спросите перед началом:**
   - Оставьте комментарий в issue, что хотите взять задачу
   - Задайте вопросы, если что-то непонятно

### Типы контрибьюций

Мы приветствуем различные виды вкладов:

- 🐛 **Исправление багов**
- ✨ **Новые фичи**
- 📝 **Улучшение документации**
- 🧪 **Добавление тестов**
- 🎨 **Улучшение UI/UX**
- ♻️ **Рефакторинг кода**
- 🌐 **Переводы**
- 💬 **Ответы на вопросы в Discussions**

---

## Настройка окружения

### Требования

- **Node.js:** 18.x или выше
- **npm:** 9.x или выше
- **PostgreSQL:** 14.x или выше
- **Redis:** 7.x или выше
- **Git:** 2.x или выше
- **Docker** (опционально): 20.x или выше

### Установка

1. **Форкните репозиторий**

   Нажмите кнопку "Fork" на GitHub

2. **Клонируйте ваш форк**

   ```bash
   git clone https://github.com/YOUR_USERNAME/rgsystem.git
   cd rgsystem
   ```

3. **Добавьте upstream remote**

   ```bash
   git remote add upstream https://github.com/original-org/rgsystem.git
   ```

4. **Установите зависимости**

   ```bash
   # Backend
   cd api
   npm install

   # Frontend
   cd ../frontend
   npm install

   # WebSocket server
   cd ../websocket
   npm install
   ```

5. **Настройте переменные окружения**

   ```bash
   cp .env.example .env
   ```

   Отредактируйте `.env` и установите свои значения:
   ```bash
   DB_HOST=localhost
   DB_PORT=5432
   DB_NAME=rgsystem_dev
   DB_USER=postgres
   DB_PASSWORD=your_password

   REDIS_HOST=localhost
   REDIS_PORT=6379
   REDIS_PASSWORD=

   JWT_SECRET=your_dev_secret_key
   ```

6. **Создайте базу данных**

   ```bash
   # Подключитесь к PostgreSQL
   psql -U postgres

   # Создайте базу данных
   CREATE DATABASE rgsystem_dev;
   \q
   ```

7. **Запустите миграции**

   ```bash
   cd api
   npm run migrate:up
   ```

8. **Заполните тестовыми данными**

   ```bash
   npm run seed
   ```

### Альтернатива: Docker

Если у вас установлен Docker:

```bash
# Запустите все сервисы
docker-compose up -d

# Проверьте статус
docker-compose ps

# Просмотр логов
docker-compose logs -f api
```

---

## Процесс разработки

### Workflow

1. **Создайте feature ветку**

   ```bash
   git checkout -b feature/my-awesome-feature
   ```

   Naming convention:
   - `feature/` — новая функциональность
   - `bugfix/` — исправление бага
   - `hotfix/` — срочное исправление
   - `docs/` — обновление документации
   - `refactor/` — рефакторинг
   - `test/` — добавление тестов

2. **Синхронизируйте с upstream**

   ```bash
   git fetch upstream
   git rebase upstream/main
   ```

3. **Пишите код**

   - Следуйте code style (см. ниже)
   - Добавляйте тесты для новой функциональности
   - Обновляйте документацию при необходимости

4. **Коммитьте изменения**

   ```bash
   git add .
   git commit -m "feat: add tiebreak algorithm for group competitions"
   ```

   Формат commit message (Conventional Commits):
   ```
   <type>(<scope>): <subject>

   <body>

   <footer>
   ```

   Types:
   - `feat:` — новая функциональность
   - `fix:` — исправление бага
   - `docs:` — обновление документации
   - `style:` — форматирование кода (без изменения логики)
   - `refactor:` — рефакторинг
   - `test:` — добавление тестов
   - `chore:` — обновление build задач, конфигов

   Примеры:
   ```
   feat(api): add endpoint for bulk athlete registration
   fix(scoring): correct E-score calculation when judges < 6
   docs(readme): update installation instructions
   test(scores): add unit tests for tiebreak algorithm
   ```

5. **Запустите тесты**

   ```bash
   npm run test
   npm run test:e2e
   npm run lint
   ```

6. **Push в ваш форк**

   ```bash
   git push origin feature/my-awesome-feature
   ```

7. **Создайте Pull Request**

   - Откройте PR из вашей feature ветки в `main` upstream репозитория
   - Заполните PR template
   - Дождитесь code review

---

## Code Style и стандарты

### JavaScript/TypeScript

Мы используем **ESLint** и **Prettier** для автоматического форматирования.

```bash
# Проверка
npm run lint

# Автоматическое исправление
npm run lint:fix

# Форматирование
npm run format
```

#### Основные правила:

- **Отступы:** 2 пробела
- **Quotes:** single quotes (')
- **Semicolons:** обязательны
- **Line length:** 100 символов
- **Naming:**
  - `camelCase` для переменных и функций
  - `PascalCase` для классов и компонентов
  - `UPPER_SNAKE_CASE` для констант
  - `kebab-case` для файлов

#### Примеры:

```javascript
// ✅ Хорошо
const calculateFinalScore = (dScore, eScore, aScore, nd) => {
  return dScore + eScore + aScore - nd;
};

class CompetitionService {
  async createCompetition(data) {
    // ...
  }
}

const MAX_ATHLETES_PER_CLUB = 10;

// ❌ Плохо
function Calculate_Final_Score(d_score,e_score,a_score,nd)
{
return d_score+e_score+a_score-nd
}
```

### SQL

- **Таблицы:** `snake_case` (lowercase)
- **Столбцы:** `snake_case` (lowercase)
- **Индексы:** `idx_table_column`
- **Constraints:** `table_column_constraint_type`

```sql
-- ✅ Хорошо
CREATE TABLE athlete_registrations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  athlete_id UUID NOT NULL REFERENCES athletes(id),
  competition_id UUID NOT NULL REFERENCES competitions(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT athlete_registrations_unique UNIQUE (athlete_id, competition_id)
);

CREATE INDEX idx_athlete_registrations_competition_id
  ON athlete_registrations(competition_id);
```

### React/Frontend

- **Components:** PascalCase, один компонент на файл
- **Hooks:** используйте функциональные компоненты
- **Props:** destructuring в параметрах
- **State:** предпочитайте хуки (`useState`, `useEffect`)

```jsx
// ✅ Хорошо
import React, { useState, useEffect } from 'react';
import PropTypes from 'prop-types';

const ScoreCard = ({ athlete, score, onEdit }) => {
  const [isEditing, setIsEditing] = useState(false);

  useEffect(() => {
    // Side effects
  }, [score]);

  return (
    <div className="score-card">
      <h3>{athlete.name}</h3>
      <div className="score">{score.final}</div>
      {isEditing && <button onClick={onEdit}>Edit</button>}
    </div>
  );
};

ScoreCard.propTypes = {
  athlete: PropTypes.object.isRequired,
  score: PropTypes.object.isRequired,
  onEdit: PropTypes.func
};

export default ScoreCard;
```

---

## Тестирование

### Типы тестов

1. **Unit Tests** — тестирование отдельных функций/модулей
2. **Integration Tests** — тестирование взаимодействия компонентов
3. **E2E Tests** — тестирование полных пользовательских сценариев

### Запуск тестов

```bash
# Все тесты
npm run test

# Unit тесты
npm run test:unit

# Integration тесты
npm run test:integration

# E2E тесты
npm run test:e2e

# С покрытием
npm run test:coverage

# Watch mode (для разработки)
npm run test:watch
```

### Написание тестов

Используем **Jest** для unit/integration тестов и **Playwright** для E2E.

```javascript
// tests/unit/scoring/calculateFinalScore.test.js
import { calculateFinalScore } from '../../../src/services/scoring';

describe('calculateFinalScore', () => {
  it('should calculate correct final score with all components', () => {
    const input = {
      d_score: 8.0,
      e_score: 9.5,
      a_score: 0.0,
      neutral_deductions: 0.1,
      penalties: 0.0
    };

    const result = calculateFinalScore(input);

    expect(result).toBe(17.4);
  });

  it('should handle neutral deductions correctly', () => {
    const input = {
      d_score: 8.0,
      e_score: 9.5,
      a_score: 0.0,
      neutral_deductions: 0.5,
      penalties: 0.2
    };

    const result = calculateFinalScore(input);

    expect(result).toBe(16.8);
  });

  it('should throw error if d_score is negative', () => {
    const input = {
      d_score: -1.0,
      e_score: 9.5,
      a_score: 0.0,
      neutral_deductions: 0.0,
      penalties: 0.0
    };

    expect(() => calculateFinalScore(input)).toThrow('D-score cannot be negative');
  });
});
```

### Test Coverage

Минимальное покрытие:
- **Statements:** 80%
- **Branches:** 75%
- **Functions:** 80%
- **Lines:** 80%

---

## Pull Request процесс

### Before Creating PR

- ✅ Код соответствует code style
- ✅ Все тесты проходят
- ✅ Покрытие тестами не ухудшилось
- ✅ Документация обновлена (если необходимо)
- ✅ Нет конфликтов с `main` веткой
- ✅ Commit messages следуют Conventional Commits

### PR Template

При создании PR, заполните template:

```markdown
## Описание

Краткое описание изменений

## Тип изменения

- [ ] Bug fix (исправление бага)
- [ ] New feature (новая функциональность)
- [ ] Breaking change (изменение, ломающее обратную совместимость)
- [ ] Documentation update (обновление документации)

## Связанные Issue

Fixes #123
Related to #456

## Как протестировано

Опишите, как вы тестировали изменения

## Checklist

- [ ] Код следует code style проекта
- [ ] Проведен self-review кода
- [ ] Добавлены комментарии для сложной логики
- [ ] Обновлена документация
- [ ] Нет новых warnings
- [ ] Добавлены тесты
- [ ] Все тесты проходят локально
- [ ] Зависимые изменения были замержены

## Screenshots (если применимо)

Добавьте скриншоты UI изменений
```

### Code Review процесс

1. **Автоматические проверки:**
   - CI/CD pipeline запускает все тесты
   - ESLint проверяет code style
   - Code coverage анализируется

2. **Human review:**
   - Минимум 1 approve от мейнтейнера
   - Все комментарии должны быть addressed

3. **Merge:**
   - Используем **Squash and merge** для фич
   - Rebase для hotfixes
   - Удаляем feature ветку после мержа

---

## Архитектурные решения

### Структура проекта

```
rgsystem/
├── api/                      # Backend API
│   ├── src/
│   │   ├── controllers/      # Request handlers
│   │   ├── services/         # Business logic
│   │   ├── models/           # Database models
│   │   ├── middleware/       # Express middleware
│   │   ├── routes/           # API routes
│   │   ├── utils/            # Helper functions
│   │   └── config/           # Configuration
│   ├── tests/
│   ├── migrations/           # Database migrations
│   └── package.json
│
├── websocket/                # WebSocket server
│   ├── src/
│   │   ├── handlers/         # Event handlers
│   │   ├── services/         # Business logic
│   │   └── config/
│   └── package.json
│
├── frontend/                 # React frontend
│   ├── src/
│   │   ├── components/       # React components
│   │   ├── pages/            # Page components
│   │   ├── hooks/            # Custom hooks
│   │   ├── services/         # API clients
│   │   ├── utils/            # Helpers
│   │   └── styles/           # CSS/SCSS
│   └── package.json
│
├── shared/                   # Shared code (types, constants)
│   ├── types/
│   └── constants/
│
├── spec_improved/            # Documentation
└── docker-compose.yml
```

### Layered Architecture

```
┌─────────────────────────────────────┐
│         Presentation Layer          │
│  (Controllers, Routes, WebSocket)   │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│         Business Logic Layer        │
│           (Services)                │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│         Data Access Layer           │
│      (Models, Repositories)         │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│           Database                  │
│       (PostgreSQL, Redis)           │
└─────────────────────────────────────┘
```

### Design Patterns

- **Repository Pattern** для доступа к данным
- **Service Layer** для бизнес-логики
- **Factory Pattern** для создания объектов
- **Observer Pattern** для WebSocket events
- **Middleware Pattern** для обработки запросов

---

## База данных

### Миграции

Используем **Knex.js** для миграций.

#### Создание миграции:

```bash
npm run migrate:make create_performances_table
```

#### Пример миграции:

```javascript
// migrations/20250115_create_performances_table.js

exports.up = async function(knex) {
  await knex.schema.createTable('performances', (table) => {
    table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    table.uuid('competition_id').notNullable()
      .references('id').inTable('competitions').onDelete('CASCADE');
    table.uuid('athlete_id').notNullable()
      .references('id').inTable('athletes').onDelete('CASCADE');
    table.string('apparatus', 20).notNullable();
    table.integer('start_order');
    table.string('status', 20).default('registered');
    table.timestamp('started_at');
    table.timestamp('completed_at');
    table.timestamps(true, true);

    table.index(['competition_id', 'apparatus']);
    table.index('athlete_id');
  });
};

exports.down = async function(knex) {
  await knex.schema.dropTableIfExists('performances');
};
```

#### Применение миграций:

```bash
# Применить все новые миграции
npm run migrate:up

# Откатить последнюю миграцию
npm run migrate:down

# Откатить все миграции
npm run migrate:rollback

# Статус миграций
npm run migrate:status
```

### Seed Data

```bash
# Заполнить тестовыми данными
npm run seed

# Конкретный seed файл
npm run seed:run 01_users.js
```

---

## API разработка

### Создание нового endpoint

1. **Определите route:**

```javascript
// api/src/routes/athletes.js
const express = require('express');
const router = express.Router();
const athleteController = require('../controllers/athleteController');
const { authenticate, authorize } = require('../middleware/auth');

router.get('/',
  authenticate,
  athleteController.list
);

router.post('/',
  authenticate,
  authorize(['organizer', 'admin']),
  athleteController.create
);

module.exports = router;
```

2. **Создайте controller:**

```javascript
// api/src/controllers/athleteController.js
const athleteService = require('../services/athleteService');
const { ValidationError } = require('../utils/errors');

exports.list = async (req, res, next) => {
  try {
    const { page = 1, limit = 20, search, club } = req.query;

    const result = await athleteService.list({
      page: parseInt(page),
      limit: parseInt(limit),
      search,
      club
    });

    res.json({
      success: true,
      data: result
    });
  } catch (error) {
    next(error);
  }
};

exports.create = async (req, res, next) => {
  try {
    const athlete = await athleteService.create(req.body);

    res.status(201).json({
      success: true,
      data: athlete
    });
  } catch (error) {
    next(error);
  }
};
```

3. **Реализуйте service:**

```javascript
// api/src/services/athleteService.js
const db = require('../config/database');
const { ValidationError, NotFoundError } = require('../utils/errors');

class AthleteService {
  async list({ page, limit, search, club }) {
    let query = db('athletes')
      .select('athletes.*', 'clubs.name as club_name')
      .leftJoin('clubs', 'athletes.club_id', 'clubs.id');

    if (search) {
      query = query.where(function() {
        this.where('athletes.first_name', 'ilike', `%${search}%`)
          .orWhere('athletes.last_name', 'ilike', `%${search}%`);
      });
    }

    if (club) {
      query = query.where('clubs.id', club);
    }

    const total = await query.clone().count('* as count').first();
    const athletes = await query
      .limit(limit)
      .offset((page - 1) * limit)
      .orderBy('athletes.last_name', 'asc');

    return {
      athletes,
      pagination: {
        page,
        limit,
        total: parseInt(total.count),
        total_pages: Math.ceil(total.count / limit)
      }
    };
  }

  async create(data) {
    // Валидация
    if (!data.first_name || !data.last_name) {
      throw new ValidationError('First name and last name are required');
    }

    // Создание
    const [athlete] = await db('athletes')
      .insert({
        first_name: data.first_name,
        last_name: data.last_name,
        middle_name: data.middle_name,
        date_of_birth: data.date_of_birth,
        club_id: data.club_id
      })
      .returning('*');

    return athlete;
  }
}

module.exports = new AthleteService();
```

4. **Добавьте тесты:**

```javascript
// api/tests/integration/athletes.test.js
const request = require('supertest');
const app = require('../../src/app');

describe('Athletes API', () => {
  let token;

  beforeAll(async () => {
    // Авторизация для получения токена
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        email: 'test@example.com',
        password: 'password123'
      });

    token = response.body.data.access_token;
  });

  describe('GET /api/athletes', () => {
    it('should return list of athletes', async () => {
      const response = await request(app)
        .get('/api/athletes')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);

      expect(response.body.success).toBe(true);
      expect(response.body.data.athletes).toBeInstanceOf(Array);
      expect(response.body.data.pagination).toBeDefined();
    });

    it('should filter athletes by search', async () => {
      const response = await request(app)
        .get('/api/athletes?search=Иванова')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);

      const athletes = response.body.data.athletes;
      athletes.forEach(athlete => {
        expect(
          athlete.first_name.includes('Иванова') ||
          athlete.last_name.includes('Иванова')
        ).toBe(true);
      });
    });
  });
});
```

---

## Frontend разработка

### Создание нового компонента

1. **Структура файлов:**

```
src/components/ScoreCard/
├── index.js
├── ScoreCard.jsx
├── ScoreCard.module.scss
└── ScoreCard.test.jsx
```

2. **Компонент:**

```jsx
// ScoreCard.jsx
import React from 'react';
import PropTypes from 'prop-types';
import styles from './ScoreCard.module.scss';

const ScoreCard = ({ athlete, score, onEdit, editable }) => {
  return (
    <div className={styles.scoreCard}>
      <div className={styles.header}>
        <span className={styles.bibNumber}>#{athlete.bib_number}</span>
        <h3 className={styles.athleteName}>
          {athlete.first_name} {athlete.last_name}
        </h3>
        <span className={styles.club}>{athlete.club}</span>
      </div>

      <div className={styles.scores}>
        <div className={styles.scoreItem}>
          <label>D</label>
          <span>{score.d_score.toFixed(2)}</span>
        </div>
        <div className={styles.scoreItem}>
          <label>E</label>
          <span>{score.e_score.toFixed(2)}</span>
        </div>
        <div className={styles.scoreItem}>
          <label>ND</label>
          <span>{score.neutral_deductions.toFixed(2)}</span>
        </div>
      </div>

      <div className={styles.finalScore}>
        <label>Final Score</label>
        <span className={styles.score}>{score.final_score.toFixed(2)}</span>
      </div>

      {editable && (
        <button className={styles.editButton} onClick={onEdit}>
          Edit Score
        </button>
      )}
    </div>
  );
};

ScoreCard.propTypes = {
  athlete: PropTypes.shape({
    id: PropTypes.string.isRequired,
    first_name: PropTypes.string.isRequired,
    last_name: PropTypes.string.isRequired,
    bib_number: PropTypes.number.isRequired,
    club: PropTypes.string.isRequired
  }).isRequired,
  score: PropTypes.shape({
    d_score: PropTypes.number.isRequired,
    e_score: PropTypes.number.isRequired,
    neutral_deductions: PropTypes.number.isRequired,
    final_score: PropTypes.number.isRequired
  }).isRequired,
  onEdit: PropTypes.func,
  editable: PropTypes.bool
};

ScoreCard.defaultProps = {
  editable: false,
  onEdit: () => {}
};

export default ScoreCard;
```

3. **Стили:**

```scss
// ScoreCard.module.scss
.scoreCard {
  background: #fff;
  border-radius: 8px;
  padding: 16px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);

  .header {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 16px;

    .bibNumber {
      font-weight: bold;
      color: #666;
    }

    .athleteName {
      flex: 1;
      margin: 0;
      font-size: 18px;
    }

    .club {
      color: #999;
      font-size: 14px;
    }
  }

  .scores {
    display: flex;
    gap: 16px;
    margin-bottom: 16px;

    .scoreItem {
      flex: 1;
      text-align: center;

      label {
        display: block;
        font-size: 12px;
        color: #666;
        margin-bottom: 4px;
      }

      span {
        font-size: 20px;
        font-weight: bold;
      }
    }
  }

  .finalScore {
    text-align: center;
    padding: 12px;
    background: #f5f5f5;
    border-radius: 4px;

    label {
      display: block;
      font-size: 14px;
      color: #666;
      margin-bottom: 4px;
    }

    .score {
      font-size: 32px;
      font-weight: bold;
      color: #2196F3;
    }
  }

  .editButton {
    width: 100%;
    margin-top: 12px;
    padding: 8px 16px;
    background: #2196F3;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;

    &:hover {
      background: #1976D2;
    }
  }
}
```

4. **Тесты:**

```jsx
// ScoreCard.test.jsx
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import ScoreCard from './ScoreCard';

describe('ScoreCard', () => {
  const mockAthlete = {
    id: '123',
    first_name: 'Анна',
    last_name: 'Иванова',
    bib_number: 45,
    club: 'Динамо'
  };

  const mockScore = {
    d_score: 8.05,
    e_score: 9.65,
    neutral_deductions: 0.05,
    final_score: 17.65
  };

  it('renders athlete information', () => {
    render(<ScoreCard athlete={mockAthlete} score={mockScore} />);

    expect(screen.getByText('#45')).toBeInTheDocument();
    expect(screen.getByText('Анна Иванова')).toBeInTheDocument();
    expect(screen.getByText('Динамо')).toBeInTheDocument();
  });

  it('renders score components', () => {
    render(<ScoreCard athlete={mockAthlete} score={mockScore} />);

    expect(screen.getByText('8.05')).toBeInTheDocument();
    expect(screen.getByText('9.65')).toBeInTheDocument();
    expect(screen.getByText('0.05')).toBeInTheDocument();
    expect(screen.getByText('17.65')).toBeInTheDocument();
  });

  it('shows edit button when editable', () => {
    render(
      <ScoreCard
        athlete={mockAthlete}
        score={mockScore}
        editable={true}
      />
    );

    expect(screen.getByText('Edit Score')).toBeInTheDocument();
  });

  it('calls onEdit when edit button clicked', () => {
    const onEditMock = jest.fn();

    render(
      <ScoreCard
        athlete={mockAthlete}
        score={mockScore}
        editable={true}
        onEdit={onEditMock}
      />
    );

    fireEvent.click(screen.getByText('Edit Score'));

    expect(onEditMock).toHaveBeenCalledTimes(1);
  });
});
```

---

## Документация

### Когда обновлять документацию

- ✅ Новый API endpoint → обновите `API_SPECIFICATION.md` и `API_EXAMPLES.md`
- ✅ Изменение схемы БД → обновите `DATABASE_SCHEMA.md`
- ✅ Новый компонент → добавьте JSDoc комментарии
- ✅ Изменение workflow → обновите `README.md`
- ✅ Новая ошибка → обновите `ERROR_HANDLING.md`

### Комментарии в коде

Используйте JSDoc для функций и классов:

```javascript
/**
 * Calculates the final score for a routine performance
 *
 * @param {Object} scores - Score components
 * @param {number} scores.d_score - Difficulty score (DB + DA + DS + DD)
 * @param {number} scores.e_score - Execution score (10.0 - deductions)
 * @param {number} scores.a_score - Artistry score (for groups only)
 * @param {number} scores.neutral_deductions - Neutral deductions (time, costume, etc.)
 * @param {number} scores.penalties - Penalties
 * @returns {number} Final score
 * @throws {ValidationError} If any score component is invalid
 *
 * @example
 * const result = calculateFinalScore({
 *   d_score: 8.0,
 *   e_score: 9.5,
 *   a_score: 0.0,
 *   neutral_deductions: 0.1,
 *   penalties: 0.0
 * });
 * // Returns: 17.4
 */
function calculateFinalScore(scores) {
  // Implementation
}
```

---

## Получение помощи

### Где задавать вопросы

- 💬 **GitHub Discussions** — общие вопросы, идеи
- 🐛 **GitHub Issues** — баги, feature requests
- 📧 **Email:** dev@rgsystem.local
- 💬 **Slack/Discord:** [ссылка на workspace]

### Полезные ресурсы

- [FIG Official Documentation](https://www.gymnastics.sport/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Express.js Guide](https://expressjs.com/)
- [React Documentation](https://react.dev/)
- [Jest Documentation](https://jestjs.io/)

---

## Code of Conduct

Мы придерживаемся [Contributor Covenant Code of Conduct](https://www.contributor-covenant.org/).

### Основные принципы:

- 🤝 Будьте уважительны и дружелюбны
- 💡 Приветствуйте разные точки зрения
- 📖 Учитесь на ошибках (своих и чужих)
- 🙏 Будьте терпеливы с новичками
- ✨ Делайте код и сообщество лучше

---

## Лицензия

Внося вклад в проект, вы соглашаетесь с тем, что ваш код будет лицензирован под той же лицензией, что и проект.

---

## Благодарности

Спасибо за ваш вклад в RG System! 🎉

Каждый Pull Request, issue, комментарий, идея делают проект лучше.

---

**Последнее обновление:** 2025-11-27
**Вопросы?** Создайте issue или напишите на dev@rgsystem.local
