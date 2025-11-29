# Development Handbook
## RG System - Rhythmic Gymnastics Competition Management System

> **Version:** 1.0
> **Date:** 2025-11-27
> **Target Audience:** Software Engineers, Tech Leads
> **Maintainer:** Development Team

---

## Table of Contents

1. [Getting Started](#1-getting-started)
2. [Architecture Overview](#2-architecture-overview)
3. [Project Structure](#3-project-structure)
4. [Coding Standards](#4-coding-standards)
5. [Development Workflow](#5-development-workflow)
6. [API Development](#6-api-development)
7. [Frontend Development](#7-frontend-development)
8. [Database Development](#8-database-development)
9. [Testing Guidelines](#9-testing-guidelines)
10. [Debugging Guide](#10-debugging-guide)
11. [Performance Optimization](#11-performance-optimization)
12. [Security Best Practices](#12-security-best-practices)
13. [Common Development Tasks](#13-common-development-tasks)
14. [Troubleshooting](#14-troubleshooting)
15. [Code Review Checklist](#15-code-review-checklist)

---

## 1. Getting Started

### 1.1. Prerequisites

**Required Software:**
```bash
# Node.js 18+ LTS
node --version  # Should be >= 18.0.0
npm --version   # Should be >= 9.0.0

# PostgreSQL 14+
psql --version  # Should be >= 14.0

# Redis 7+
redis-cli --version  # Should be >= 7.0

# Docker & Docker Compose (optional but recommended)
docker --version
docker-compose --version

# Git
git --version
```

**Recommended Tools:**
- **IDE:** VS Code, WebStorm, or IntelliJ IDEA
- **Database GUI:** DBeaver, pgAdmin, or TablePlus
- **API Client:** Postman, Insomnia, or REST Client (VS Code extension)
- **Git GUI:** GitKraken, SourceTree, or built-in IDE Git

### 1.2. Local Development Setup

**Step 1: Clone Repository**
```bash
git clone https://github.com/your-org/rg-system.git
cd rg-system
```

**Step 2: Install Dependencies**
```bash
# Install backend dependencies
cd backend
npm install

# Install frontend dependencies
cd ../frontend
npm install
```

**Step 3: Setup Environment Variables**
```bash
# Copy example env file
cp .env.example .env

# Edit .env with your local configuration
nano .env
```

**Example `.env` for local development:**
```bash
# Application
NODE_ENV=development
PORT=3000
LOG_LEVEL=debug

# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/rgsystem_dev
DB_POOL_MIN=2
DB_POOL_MAX=10

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=your-super-secret-jwt-key-change-in-production
JWT_ACCESS_EXPIRY=15m
JWT_REFRESH_EXPIRY=7d

# CORS
CORS_ORIGIN=http://localhost:3001

# WebSocket
WS_PORT=3002
```

**Step 4: Setup Database**
```bash
# Create database
createdb rgsystem_dev

# Run migrations
npm run db:migrate

# Seed database with test data (optional)
npm run db:seed
```

**Step 5: Start Development Servers**
```bash
# Terminal 1: Backend API
cd backend
npm run dev  # Starts on http://localhost:3000

# Terminal 2: Frontend
cd frontend
npm run dev  # Starts on http://localhost:3001

# Terminal 3: WebSocket Server
cd backend
npm run dev:ws  # Starts on ws://localhost:3002
```

**Step 6: Verify Setup**
```bash
# Check backend health
curl http://localhost:3000/health

# Expected response:
# {"status":"ok","database":"connected","redis":"connected"}
```

### 1.3. Docker Setup (Alternative)

**Quick start with Docker Compose:**
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop all services
docker-compose down

# Rebuild after code changes
docker-compose up -d --build
```

**Access services:**
- API: http://localhost:3000
- Frontend: http://localhost:3001
- WebSocket: ws://localhost:3002
- PostgreSQL: localhost:5432
- Redis: localhost:6379
- Adminer (DB GUI): http://localhost:8080

### 1.4. VS Code Setup

**Recommended Extensions:**
```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-typescript-next",
    "bradlc.vscode-tailwindcss",
    "prisma.prisma",
    "humao.rest-client",
    "eamodio.gitlens",
    "streetsidesoftware.code-spell-checker",
    "usernamehw.errorlens",
    "ms-azuretools.vscode-docker"
  ]
}
```

**Workspace Settings (`.vscode/settings.json`):**
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "files.exclude": {
    "**/.git": true,
    "**/.DS_Store": true,
    "**/node_modules": true,
    "**/dist": true,
    "**/build": true
  }
}
```

---

## 2. Architecture Overview

### 2.1. System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         RG System Architecture                   │
└─────────────────────────────────────────────────────────────────┘

┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Judge      │     │  Secretary   │     │   Viewer     │
│   Client     │     │   Client     │     │   Client     │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                   ┌────────▼────────┐
                   │  Nginx (Proxy)  │
                   │  Load Balancer  │
                   └────────┬────────┘
                            │
          ┌─────────────────┴─────────────────┐
          │                                   │
    ┌─────▼──────┐                    ┌──────▼──────┐
    │  REST API  │                    │  WebSocket  │
    │  (Express) │◄───────────────────┤   Server    │
    └─────┬──────┘                    └──────┬──────┘
          │                                   │
    ┌─────▼──────────────────────────────────▼─────┐
    │              Service Layer                    │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
    │  │ Scoring  │  │  Auth    │  │ Notify   │   │
    │  │ Service  │  │ Service  │  │ Service  │   │
    │  └──────────┘  └──────────┘  └──────────┘   │
    └───────────────────┬──────────────────────────┘
                        │
          ┌─────────────┴─────────────┐
          │                           │
    ┌─────▼─────┐              ┌──────▼──────┐
    │PostgreSQL │              │    Redis    │
    │    14     │              │     7       │
    │(Primary + │              │  (Cache +   │
    │ Replica)  │              │  Sessions)  │
    └───────────┘              └─────────────┘
```

### 2.2. Technology Stack

**Backend:**
- **Runtime:** Node.js 18 LTS
- **Framework:** Express.js 4.18+
- **Language:** TypeScript 5.0+
- **ORM:** Prisma 5.0+ or Knex.js
- **Validation:** Joi or Zod
- **Authentication:** Passport.js + JWT
- **WebSocket:** Socket.IO 4.0+
- **Task Queue:** Bull (Redis-based)

**Frontend:**
- **Framework:** React 18+
- **Language:** TypeScript 5.0+
- **State Management:** Zustand or Redux Toolkit
- **Routing:** React Router 6+
- **UI Framework:** Tailwind CSS + shadcn/ui
- **Forms:** React Hook Form + Zod
- **API Client:** Axios or TanStack Query
- **Build Tool:** Vite 4.0+

**Database:**
- **Primary:** PostgreSQL 14+
- **Cache:** Redis 7+
- **Migrations:** Prisma Migrate or Knex migrations

**DevOps:**
- **Containerization:** Docker + Docker Compose
- **Reverse Proxy:** Nginx
- **CI/CD:** GitHub Actions
- **Monitoring:** Prometheus + Grafana
- **Logging:** Winston + ELK Stack

### 2.3. Design Patterns

**Layered Architecture:**

```
┌─────────────────────────────────────────────┐
│          Presentation Layer                  │
│  (Controllers, Routes, Middleware)           │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│          Business Logic Layer                │
│  (Services, Domain Logic, Validators)        │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│          Data Access Layer                   │
│  (Repositories, Models, Database)            │
└─────────────────────────────────────────────┘
```

**Key Patterns Used:**

1. **Repository Pattern** - Abstraction over data access
2. **Service Layer Pattern** - Business logic encapsulation
3. **Dependency Injection** - Loose coupling
4. **Factory Pattern** - Object creation (test data, DTOs)
5. **Strategy Pattern** - Scoring algorithms (D-score, E-score)
6. **Observer Pattern** - WebSocket event notifications
7. **Middleware Pattern** - Request processing pipeline
8. **DTO Pattern** - Data transfer objects for API responses

### 2.4. Key Architectural Decisions

**ADR-001: Use TypeScript Instead of JavaScript**
- **Decision:** All new code written in TypeScript
- **Rationale:** Type safety, better IDE support, fewer runtime errors
- **Status:** Accepted
- **Date:** 2025-01-15

**ADR-002: Use Prisma as ORM**
- **Decision:** Prisma for database access instead of raw SQL or Sequelize
- **Rationale:** Type-safe queries, excellent TypeScript support, modern migrations
- **Status:** Accepted
- **Date:** 2025-01-20

**ADR-003: WebSocket for Real-Time Score Updates**
- **Decision:** Socket.IO for real-time communication
- **Rationale:** Bi-directional communication, fallback to polling, room-based broadcasting
- **Status:** Accepted
- **Date:** 2025-01-25

**ADR-004: Monorepo vs Multi-Repo**
- **Decision:** Monorepo with separate backend/ and frontend/ directories
- **Rationale:** Easier code sharing, atomic commits across stack, simpler CI/CD
- **Status:** Accepted
- **Date:** 2025-02-01

---

## 3. Project Structure

### 3.1. Backend Structure

```
backend/
├── src/
│   ├── config/              # Configuration files
│   │   ├── database.ts      # Database connection config
│   │   ├── redis.ts         # Redis config
│   │   └── jwt.ts           # JWT config
│   │
│   ├── controllers/         # Route controllers (thin layer)
│   │   ├── auth.controller.ts
│   │   ├── competitions.controller.ts
│   │   ├── athletes.controller.ts
│   │   ├── scores.controller.ts
│   │   └── judges.controller.ts
│   │
│   ├── services/            # Business logic
│   │   ├── auth/
│   │   │   ├── auth.service.ts
│   │   │   ├── jwt.service.ts
│   │   │   └── password.service.ts
│   │   ├── scoring/
│   │   │   ├── d-score.service.ts
│   │   │   ├── e-score.service.ts
│   │   │   ├── final-score.service.ts
│   │   │   └── tiebreak.service.ts
│   │   ├── competition.service.ts
│   │   ├── athlete.service.ts
│   │   └── notification.service.ts
│   │
│   ├── repositories/        # Data access layer
│   │   ├── competition.repository.ts
│   │   ├── athlete.repository.ts
│   │   ├── score.repository.ts
│   │   └── user.repository.ts
│   │
│   ├── models/              # Database models (Prisma schema)
│   │   └── schema.prisma
│   │
│   ├── middleware/          # Express middleware
│   │   ├── auth.middleware.ts
│   │   ├── validation.middleware.ts
│   │   ├── error.middleware.ts
│   │   ├── logging.middleware.ts
│   │   └── rate-limit.middleware.ts
│   │
│   ├── validators/          # Request validation schemas
│   │   ├── auth.validator.ts
│   │   ├── competition.validator.ts
│   │   ├── score.validator.ts
│   │   └── common.validator.ts
│   │
│   ├── routes/              # API routes
│   │   ├── index.ts         # Route aggregator
│   │   ├── auth.routes.ts
│   │   ├── competitions.routes.ts
│   │   ├── athletes.routes.ts
│   │   └── scores.routes.ts
│   │
│   ├── dto/                 # Data Transfer Objects
│   │   ├── auth.dto.ts
│   │   ├── competition.dto.ts
│   │   ├── score.dto.ts
│   │   └── response.dto.ts
│   │
│   ├── types/               # TypeScript type definitions
│   │   ├── express.d.ts     # Express augmentation
│   │   ├── score.types.ts
│   │   └── competition.types.ts
│   │
│   ├── utils/               # Utility functions
│   │   ├── logger.ts
│   │   ├── errors.ts
│   │   ├── crypto.ts
│   │   └── date.ts
│   │
│   ├── websocket/           # WebSocket server
│   │   ├── server.ts
│   │   ├── handlers/
│   │   │   ├── score.handler.ts
│   │   │   └── competition.handler.ts
│   │   └── rooms.ts
│   │
│   ├── jobs/                # Background jobs
│   │   ├── email.job.ts
│   │   ├── backup.job.ts
│   │   └── cleanup.job.ts
│   │
│   ├── app.ts               # Express app setup
│   ├── server.ts            # HTTP server entry point
│   └── ws-server.ts         # WebSocket server entry point
│
├── tests/                   # Tests
│   ├── unit/
│   │   ├── services/
│   │   └── utils/
│   ├── integration/
│   │   ├── api/
│   │   └── database/
│   └── e2e/
│       └── scenarios/
│
├── migrations/              # Database migrations
│   └── 001_initial_schema.sql
│
├── prisma/                  # Prisma configuration
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
│
├── scripts/                 # Utility scripts
│   ├── seed-data.ts
│   ├── create-admin.ts
│   └── backup-db.sh
│
├── .env.example
├── .eslintrc.js
├── .prettierrc
├── tsconfig.json
├── package.json
└── docker-compose.yml
```

### 3.2. Frontend Structure

```
frontend/
├── src/
│   ├── components/          # React components
│   │   ├── common/          # Shared components
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Modal.tsx
│   │   │   └── Table.tsx
│   │   ├── layout/          # Layout components
│   │   │   ├── Header.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   └── Footer.tsx
│   │   ├── auth/            # Auth-related components
│   │   │   ├── LoginForm.tsx
│   │   │   └── ProtectedRoute.tsx
│   │   ├── competition/     # Competition components
│   │   │   ├── CompetitionList.tsx
│   │   │   ├── CompetitionForm.tsx
│   │   │   └── CompetitionCard.tsx
│   │   ├── scoring/         # Scoring components
│   │   │   ├── ScoreEntryForm.tsx
│   │   │   ├── ScoreDisplay.tsx
│   │   │   └── FinalResults.tsx
│   │   └── athlete/         # Athlete components
│   │       ├── AthleteList.tsx
│   │       └── AthleteProfile.tsx
│   │
│   ├── pages/               # Page components (routes)
│   │   ├── Home.tsx
│   │   ├── Login.tsx
│   │   ├── Dashboard.tsx
│   │   ├── Competitions.tsx
│   │   ├── JudgingPanel.tsx
│   │   ├── SecretaryView.tsx
│   │   └── PublicResults.tsx
│   │
│   ├── hooks/               # Custom React hooks
│   │   ├── useAuth.ts
│   │   ├── useWebSocket.ts
│   │   ├── useScores.ts
│   │   └── useCompetitions.ts
│   │
│   ├── store/               # State management (Zustand)
│   │   ├── authStore.ts
│   │   ├── competitionStore.ts
│   │   ├── scoreStore.ts
│   │   └── index.ts
│   │
│   ├── services/            # API services
│   │   ├── api.ts           # Axios instance
│   │   ├── auth.service.ts
│   │   ├── competition.service.ts
│   │   ├── score.service.ts
│   │   └── websocket.service.ts
│   │
│   ├── types/               # TypeScript types
│   │   ├── auth.types.ts
│   │   ├── competition.types.ts
│   │   ├── score.types.ts
│   │   └── common.types.ts
│   │
│   ├── utils/               # Utility functions
│   │   ├── formatting.ts
│   │   ├── validation.ts
│   │   ├── storage.ts       # LocalStorage helpers
│   │   └── constants.ts
│   │
│   ├── styles/              # Global styles
│   │   ├── globals.css
│   │   └── tailwind.css
│   │
│   ├── App.tsx              # Root component
│   ├── main.tsx             # Entry point
│   └── router.tsx           # Route configuration
│
├── public/                  # Static assets
│   ├── logo.svg
│   └── favicon.ico
│
├── tests/                   # Tests
│   ├── components/
│   ├── hooks/
│   └── utils/
│
├── .env.example
├── .eslintrc.cjs
├── .prettierrc
├── tsconfig.json
├── vite.config.ts
├── tailwind.config.js
├── package.json
└── index.html
```

### 3.3. Naming Conventions

**Files:**
- **Components:** PascalCase (`CompetitionList.tsx`, `ScoreEntryForm.tsx`)
- **Services:** camelCase with `.service.ts` suffix (`auth.service.ts`)
- **Utils:** camelCase with `.ts` suffix (`logger.ts`, `crypto.ts`)
- **Types:** camelCase with `.types.ts` suffix (`score.types.ts`)
- **Tests:** Same as source file with `.test.ts` or `.spec.ts` suffix

**Variables & Functions:**
- **Variables:** camelCase (`userName`, `scoreValue`)
- **Constants:** UPPER_SNAKE_CASE (`MAX_SCORE`, `API_BASE_URL`)
- **Functions:** camelCase, verb-first (`getUserById`, `calculateScore`)
- **Classes:** PascalCase (`CompetitionService`, `ScoreRepository`)
- **Interfaces:** PascalCase, prefix with `I` optional (`User` or `IUser`)
- **Types:** PascalCase (`ScoreType`, `CompetitionStatus`)
- **Enums:** PascalCase (`Role`, `ScoreType`)

**Database:**
- **Tables:** snake_case, plural (`competitions`, `athlete_registrations`)
- **Columns:** snake_case (`first_name`, `created_at`)
- **Indexes:** `idx_<table>_<columns>` (`idx_scores_start_list_id`)
- **Foreign Keys:** `fk_<table>_<ref_table>` (`fk_scores_athletes`)

---

## 4. Coding Standards

### 4.1. TypeScript Style Guide

**General Rules:**
```typescript
// ✅ GOOD: Use const for variables that won't be reassigned
const maxScore = 10;

// ❌ BAD: Using let when const is appropriate
let maxScore = 10;

// ✅ GOOD: Use let for variables that will be reassigned
let currentScore = 0;
currentScore = 8.5;

// ❌ BAD: Never use var
var score = 5; // NEVER DO THIS

// ✅ GOOD: Explicit return types for functions
function calculateDScore(routine: Routine): number {
  return routine.bodyDifficulties.reduce((sum, val) => sum + val, 0);
}

// ❌ BAD: Missing return type
function calculateDScore(routine: Routine) {
  return routine.bodyDifficulties.reduce((sum, val) => sum + val, 0);
}

// ✅ GOOD: Use interfaces for object shapes
interface User {
  id: string;
  email: string;
  role: Role;
}

// ✅ GOOD: Use type for unions, primitives, tuples
type ScoreType = 'D' | 'E' | 'A';
type Coordinate = [number, number];

// ✅ GOOD: Use enum for fixed set of values
enum Role {
  Judge = 'judge',
  Secretary = 'secretary',
  Organizer = 'organizer'
}

// ✅ GOOD: Async/await instead of promises
async function getUser(id: string): Promise<User> {
  const user = await userRepository.findById(id);
  return user;
}

// ❌ BAD: Promise chaining when async/await is clearer
function getUser(id: string): Promise<User> {
  return userRepository.findById(id).then(user => user);
}
```

**Strict Type Checking:**
```typescript
// tsconfig.json - Enable strict mode
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true
  }
}

// ✅ GOOD: Handle null/undefined explicitly
function getUserName(user: User | null): string {
  if (!user) {
    return 'Guest';
  }
  return user.name;
}

// ❌ BAD: Ignoring potential null
function getUserName(user: User | null): string {
  return user!.name; // Using ! (non-null assertion) is risky
}

// ✅ GOOD: Use optional chaining
const athleteName = competition?.athletes?.[0]?.name ?? 'Unknown';

// ❌ BAD: Manual null checks
const athleteName =
  competition &&
  competition.athletes &&
  competition.athletes[0] &&
  competition.athletes[0].name
    ? competition.athletes[0].name
    : 'Unknown';
```

### 4.2. ESLint Configuration

**.eslintrc.js:**
```javascript
module.exports = {
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:@typescript-eslint/recommended-requiring-type-checking',
    'prettier' // Must be last
  ],
  parser: '@typescript-eslint/parser',
  parserOptions: {
    project: './tsconfig.json',
    tsconfigRootDir: __dirname,
  },
  plugins: ['@typescript-eslint', 'import'],
  rules: {
    // TypeScript-specific rules
    '@typescript-eslint/explicit-function-return-type': 'error',
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-unused-vars': ['error', {
      argsIgnorePattern: '^_',
      varsIgnorePattern: '^_'
    }],
    '@typescript-eslint/naming-convention': [
      'error',
      {
        selector: 'interface',
        format: ['PascalCase']
      },
      {
        selector: 'typeAlias',
        format: ['PascalCase']
      },
      {
        selector: 'enum',
        format: ['PascalCase']
      }
    ],

    // Import rules
    'import/order': ['error', {
      'groups': [
        'builtin',
        'external',
        'internal',
        'parent',
        'sibling',
        'index'
      ],
      'newlines-between': 'always',
      'alphabetize': {
        'order': 'asc',
        'caseInsensitive': true
      }
    }],

    // General rules
    'no-console': ['warn', { allow: ['warn', 'error'] }],
    'prefer-const': 'error',
    'no-var': 'error',
    'eqeqeq': ['error', 'always'],
    'curly': ['error', 'all'],
    'arrow-body-style': ['error', 'as-needed']
  }
};
```

### 4.3. Prettier Configuration

**.prettierrc:**
```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "arrowParens": "avoid",
  "endOfLine": "lf"
}
```

### 4.4. Error Handling

**Use Custom Error Classes:**
```typescript
// utils/errors.ts
export class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number,
    public code: string,
    public isOperational = true
  ) {
    super(message);
    Error.captureStackTrace(this, this.constructor);
  }
}

export class ValidationError extends AppError {
  constructor(message: string, details?: Record<string, unknown>) {
    super(message, 400, 'VALIDATION_ERROR');
    this.details = details;
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`, 404, 'NOT_FOUND');
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super(message, 401, 'UNAUTHORIZED');
  }
}

// ✅ GOOD: Use custom errors
if (!competition) {
  throw new NotFoundError('Competition', competitionId);
}

if (score < 0 || score > 10) {
  throw new ValidationError('Score must be between 0 and 10', {
    field: 'score',
    value: score
  });
}

// ❌ BAD: Generic errors
if (!competition) {
  throw new Error('Not found');
}
```

**Try-Catch Best Practices:**
```typescript
// ✅ GOOD: Specific error handling
async function submitScore(scoreData: ScoreInput): Promise<Score> {
  try {
    const validatedData = validateScore(scoreData);
    const score = await scoreRepository.create(validatedData);
    await notificationService.notifyScoreSubmitted(score.id);
    return score;
  } catch (error) {
    if (error instanceof ValidationError) {
      logger.warn('Score validation failed', { scoreData, error });
      throw error; // Re-throw to be handled by controller
    }

    if (error instanceof DatabaseError) {
      logger.error('Database error when submitting score', { error });
      throw new AppError('Failed to submit score', 500, 'DB_ERROR');
    }

    logger.error('Unexpected error in submitScore', { error });
    throw new AppError('An unexpected error occurred', 500, 'INTERNAL_ERROR');
  }
}

// ❌ BAD: Catch and ignore
async function submitScore(scoreData: ScoreInput): Promise<Score | null> {
  try {
    return await scoreRepository.create(scoreData);
  } catch (error) {
    console.log('Error:', error);
    return null; // Swallowing errors is dangerous!
  }
}
```

### 4.5. Comments & Documentation

**JSDoc for Public APIs:**
```typescript
/**
 * Calculates the final D-score for an individual routine
 * according to FIG 2025-2028 Code of Points.
 *
 * @param routine - The routine to score
 * @returns The calculated D-score (sum of DB, DA, DS, DD)
 * @throws {ValidationError} If routine has fewer than 4 body difficulties
 *
 * @example
 * ```typescript
 * const routine = {
 *   bodyDifficulties: [0.3, 0.5, 0.6, 0.7, 0.8],
 *   apparatusDifficulties: [0.2, 0.3],
 *   danceSteps: [0.1, 0.2],
 *   dynamicRotations: [0.1, 0.2, 0.3]
 * };
 * const dScore = calculateDScore(routine); // Returns 4.4
 * ```
 */
export function calculateDScore(routine: IndividualRoutine): number {
  // Implementation
}

// ✅ GOOD: Comments explain WHY, not WHAT
// Use average of middle 2 E-scores to eliminate outliers (FIG rules)
const middleScores = eScores.slice(1, -1);
const average = middleScores.reduce((sum, val) => sum + val, 0) / middleScores.length;

// ❌ BAD: Comments that just repeat the code
// Loop through scores and add them up, then divide by length
const average = scores.reduce((sum, val) => sum + val, 0) / scores.length;
```

**TODO Comments:**
```typescript
// ✅ GOOD: Actionable TODOs with context
// TODO(john): Implement caching for competition list (expected perf improvement: 2x)
// Related ticket: JIRA-123

// TODO: URGENT - Fix memory leak in WebSocket handler before v2.0 release
// Reproduces after ~1000 concurrent connections

// ❌ BAD: Vague TODOs
// TODO: fix this
// TODO: improve performance
```

---

## 5. Development Workflow

### 5.1. Git Workflow (Git Flow)

**Branch Naming:**
```
main              # Production-ready code
├── develop       # Integration branch for features
├── feature/*     # New features
│   ├── feature/scoring-algorithm
│   └── feature/judge-dashboard
├── bugfix/*      # Bug fixes for develop
│   └── bugfix/websocket-reconnect
├── hotfix/*      # Urgent production fixes
│   └── hotfix/security-patch-jwt
└── release/*     # Release preparation
    └── release/v2.5.0
```

**Standard Workflow:**
```bash
# 1. Start new feature
git checkout develop
git pull origin develop
git checkout -b feature/athlete-registration

# 2. Make changes, commit often
git add .
git commit -m "feat(athletes): add registration form validation"

# 3. Keep feature branch up to date
git checkout develop
git pull origin develop
git checkout feature/athlete-registration
git rebase develop

# 4. Push to remote
git push origin feature/athlete-registration

# 5. Create Pull Request on GitHub
# - Add description
# - Link related issues
# - Request reviewers

# 6. After PR approval, merge to develop
# (Done via GitHub UI with squash merge)

# 7. Delete feature branch
git branch -d feature/athlete-registration
git push origin --delete feature/athlete-registration
```

### 5.2. Commit Message Convention

**Format:** `<type>(<scope>): <subject>`

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation only
- `style`: Code style changes (formatting, missing semicolons, etc.)
- `refactor`: Code refactoring (no functional changes)
- `perf`: Performance improvements
- `test`: Adding/updating tests
- `chore`: Build process, dependencies, tooling

**Examples:**
```bash
# ✅ GOOD
git commit -m "feat(scoring): implement D-score calculation algorithm"
git commit -m "fix(auth): resolve JWT token expiration issue"
git commit -m "docs(api): update competition endpoints documentation"
git commit -m "test(scoring): add unit tests for E-score calculation"
git commit -m "perf(database): add index on scores.start_list_id"
git commit -m "refactor(services): extract scoring logic to separate service"

# ❌ BAD
git commit -m "fix bug"
git commit -m "update code"
git commit -m "WIP"
git commit -m "asdfasdf"
```

**Multi-line commits:**
```bash
git commit -m "feat(competitions): add competition filtering

- Add filter by status (draft, active, completed)
- Add filter by date range
- Add filter by organizer
- Implement server-side pagination

Closes #123"
```

### 5.3. Pull Request Process

**PR Template:**
```markdown
## Description
<!-- Brief description of what this PR does -->

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Related Issues
Closes #123
Related to #456

## Changes Made
- Added D-score calculation algorithm
- Implemented validation for score range (0-10)
- Added unit tests for scoring service

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed
- [ ] All existing tests pass

### Manual Testing Steps:
1. Navigate to judging panel
2. Enter D-score components
3. Verify total D-score is calculated correctly

## Screenshots (if applicable)
<!-- Add screenshots here -->

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review performed
- [ ] Code commented where necessary
- [ ] Documentation updated
- [ ] No new warnings introduced
- [ ] Tests added that prove fix/feature works
- [ ] Dependent changes merged

## Additional Notes
<!-- Any additional information -->
```

**Review Checklist (for Reviewers):**
- [ ] Code follows TypeScript/React best practices
- [ ] No console.log statements (use logger)
- [ ] Error handling is comprehensive
- [ ] Input validation is present
- [ ] SQL injection / XSS vulnerabilities checked
- [ ] Tests added for new functionality
- [ ] Tests pass (run `npm test`)
- [ ] No TypeScript errors (`npm run typecheck`)
- [ ] ESLint passes (`npm run lint`)
- [ ] API documentation updated if endpoints changed
- [ ] Database migrations included if schema changed

---

## 6. API Development

### 6.1. RESTful API Design Principles

**Resource Naming:**
```bash
# ✅ GOOD: Plural nouns, hierarchical
GET    /api/competitions                    # List competitions
GET    /api/competitions/:id                # Get competition
POST   /api/competitions                    # Create competition
PUT    /api/competitions/:id                # Update competition
DELETE /api/competitions/:id                # Delete competition

GET    /api/competitions/:id/athletes       # List athletes in competition
POST   /api/competitions/:id/athletes       # Register athlete
GET    /api/competitions/:id/scores         # Get all scores

# ❌ BAD: Verbs in URLs, singular
GET    /api/getCompetition/:id
POST   /api/createCompetition
GET    /api/competition                     # Should be plural
```

**HTTP Status Codes:**
```typescript
// 2xx Success
200 OK                  // GET, PUT successful
201 Created             // POST successful
204 No Content          // DELETE successful

// 4xx Client Errors
400 Bad Request         // Validation error
401 Unauthorized        // Not authenticated
403 Forbidden           // Not authorized
404 Not Found           // Resource doesn't exist
409 Conflict            // Duplicate, constraint violation
422 Unprocessable       // Semantic validation error

// 5xx Server Errors
500 Internal Server Error   // Unexpected server error
502 Bad Gateway             // External service error
503 Service Unavailable     // Temporary unavailable
```

### 6.2. Controller Pattern

**Example Controller:**
```typescript
// controllers/competitions.controller.ts
import { Request, Response, NextFunction } from 'express';
import { CompetitionService } from '../services/competition.service';
import { CreateCompetitionDTO, UpdateCompetitionDTO } from '../dto/competition.dto';
import { logger } from '../utils/logger';

export class CompetitionController {
  constructor(private competitionService: CompetitionService) {}

  /**
   * GET /api/competitions
   * List all competitions with pagination and filters
   */
  async list(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { page = 1, limit = 20, status, organizerId } = req.query;

      const result = await this.competitionService.list({
        page: Number(page),
        limit: Number(limit),
        status: status as string,
        organizerId: organizerId as string,
      });

      res.status(200).json({
        success: true,
        data: result.competitions,
        meta: {
          total: result.total,
          page: result.page,
          limit: result.limit,
          totalPages: Math.ceil(result.total / result.limit),
        },
      });
    } catch (error) {
      next(error); // Pass to error middleware
    }
  }

  /**
   * GET /api/competitions/:id
   * Get competition by ID
   */
  async getById(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { id } = req.params;

      const competition = await this.competitionService.findById(id);

      res.status(200).json({
        success: true,
        data: competition,
      });
    } catch (error) {
      next(error);
    }
  }

  /**
   * POST /api/competitions
   * Create new competition
   */
  async create(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const dto: CreateCompetitionDTO = req.body;
      const userId = req.user!.id; // From auth middleware

      const competition = await this.competitionService.create(dto, userId);

      logger.info('Competition created', { competitionId: competition.id, userId });

      res.status(201).json({
        success: true,
        data: competition,
      });
    } catch (error) {
      next(error);
    }
  }

  /**
   * PUT /api/competitions/:id
   * Update competition
   */
  async update(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { id } = req.params;
      const dto: UpdateCompetitionDTO = req.body;
      const userId = req.user!.id;

      const competition = await this.competitionService.update(id, dto, userId);

      res.status(200).json({
        success: true,
        data: competition,
      });
    } catch (error) {
      next(error);
    }
  }

  /**
   * DELETE /api/competitions/:id
   * Delete competition (soft delete)
   */
  async delete(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { id } = req.params;
      const userId = req.user!.id;

      await this.competitionService.delete(id, userId);

      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }
}
```

### 6.3. Service Layer Pattern

**Example Service:**
```typescript
// services/competition.service.ts
import { CompetitionRepository } from '../repositories/competition.repository';
import { CreateCompetitionDTO, UpdateCompetitionDTO } from '../dto/competition.dto';
import { NotFoundError, ForbiddenError } from '../utils/errors';
import { logger } from '../utils/logger';

export class CompetitionService {
  constructor(private competitionRepository: CompetitionRepository) {}

  async list(filters: {
    page: number;
    limit: number;
    status?: string;
    organizerId?: string;
  }): Promise<{ competitions: Competition[]; total: number; page: number; limit: number }> {
    const { page, limit, status, organizerId } = filters;

    const [competitions, total] = await Promise.all([
      this.competitionRepository.findAll({
        skip: (page - 1) * limit,
        take: limit,
        where: {
          ...(status && { status }),
          ...(organizerId && { organizerId }),
        },
        include: {
          organizer: {
            select: { id: true, name: true, email: true },
          },
        },
      }),
      this.competitionRepository.count({
        where: {
          ...(status && { status }),
          ...(organizerId && { organizerId }),
        },
      }),
    ]);

    return { competitions, total, page, limit };
  }

  async findById(id: string): Promise<Competition> {
    const competition = await this.competitionRepository.findById(id, {
      include: {
        organizer: true,
        athletes: true,
        events: true,
      },
    });

    if (!competition) {
      throw new NotFoundError('Competition', id);
    }

    return competition;
  }

  async create(dto: CreateCompetitionDTO, userId: string): Promise<Competition> {
    // Validate dates
    if (new Date(dto.end_date) < new Date(dto.start_date)) {
      throw new ValidationError('End date must be after start date');
    }

    const competition = await this.competitionRepository.create({
      ...dto,
      organizer_id: userId,
      status: 'draft',
    });

    logger.info('Competition created', { competitionId: competition.id, userId });

    return competition;
  }

  async update(
    id: string,
    dto: UpdateCompetitionDTO,
    userId: string
  ): Promise<Competition> {
    const existing = await this.findById(id);

    // Authorization check
    if (existing.organizer_id !== userId) {
      throw new ForbiddenError('You are not authorized to update this competition');
    }

    // Business rule: Can't update active competition dates
    if (existing.status === 'active' && (dto.start_date || dto.end_date)) {
      throw new ValidationError('Cannot change dates of an active competition');
    }

    const updated = await this.competitionRepository.update(id, dto);

    logger.info('Competition updated', { competitionId: id, userId });

    return updated;
  }

  async delete(id: string, userId: string): Promise<void> {
    const existing = await this.findById(id);

    if (existing.organizer_id !== userId) {
      throw new ForbiddenError('You are not authorized to delete this competition');
    }

    // Soft delete
    await this.competitionRepository.update(id, { deleted_at: new Date() });

    logger.info('Competition deleted', { competitionId: id, userId });
  }
}
```

### 6.4. Input Validation

**Validation with Zod:**
```typescript
// validators/competition.validator.ts
import { z } from 'zod';

export const createCompetitionSchema = z.object({
  body: z.object({
    name_ru: z.string().min(3).max(200),
    name_en: z.string().min(3).max(200),
    start_date: z.string().datetime(),
    end_date: z.string().datetime(),
    venue: z.string().min(3).max(200),
    description: z.string().max(2000).optional(),
    max_participants: z.number().int().positive().max(1000).optional(),
  }),
});

export const updateCompetitionSchema = z.object({
  params: z.object({
    id: z.string().uuid(),
  }),
  body: z.object({
    name_ru: z.string().min(3).max(200).optional(),
    name_en: z.string().min(3).max(200).optional(),
    start_date: z.string().datetime().optional(),
    end_date: z.string().datetime().optional(),
    venue: z.string().min(3).max(200).optional(),
    description: z.string().max(2000).optional(),
    status: z.enum(['draft', 'active', 'completed', 'archived']).optional(),
  }),
});

// Validation middleware
import { Request, Response, NextFunction } from 'express';
import { AnyZodObject, ZodError } from 'zod';

export const validate = (schema: AnyZodObject) => {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      await schema.parseAsync({
        body: req.body,
        query: req.query,
        params: req.params,
      });
      next();
    } catch (error) {
      if (error instanceof ZodError) {
        res.status(400).json({
          success: false,
          error: {
            code: 'VALIDATION_ERROR',
            message: 'Invalid request data',
            details: error.errors.map(err => ({
              field: err.path.join('.'),
              message: err.message,
            })),
          },
        });
        return;
      }
      next(error);
    }
  };
};

// Usage in routes
router.post(
  '/competitions',
  authenticate,
  validate(createCompetitionSchema),
  competitionController.create
);
```

---

(Due to length constraints, this is Part 1 of the Development Handbook. The document continues with sections 7-15 covering Frontend Development, Database Development, Testing, Debugging, Performance, Security, Common Tasks, Troubleshooting, and Code Review)

---

**To be continued in Part 2...**

This handbook is a living document. Please contribute improvements via pull requests.

**Last Updated:** 2025-11-27
**Version:** 1.0
**Maintainers:** Development Team
