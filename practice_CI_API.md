# Практическая работа. CI-пайплайн, описание API в OpenAPI и контрактные тесты

**Дисциплина:** «Технология разработки программного обеспечения»
**Тема:** Построение CI-пайплайна (GitHub Actions / GitLab CI). Контрактное тестирование и API
**Формат:** работа в парах (или индивидуально)
**Время:** 4 академических часа: часть А (CI) — 2 часа, часть Б (API и контракты) — 2 часа
**Инструменты:** Git, GitHub, GitHub Actions, Node.js (LTS, v20+), npm, Express, Jest, Supertest, ESLint, OpenAPI 3.0.3, Spectral, Prism, `jest-openapi`

---

## Содержание

1. [Цель и результаты](#1-цель-и-результаты)
2. [Подготовка](#2-подготовка)
3. [Легенда проекта](#3-легенда-проекта)

**Часть А. CI-пайплайн**

4. [Шаг 1. Репозиторий](#шаг-1-репозиторий)
5. [Шаг 2. Учебное приложение](#шаг-2-учебное-приложение)
6. [Шаг 3. Скрипты, ESLint и тесты локально](#шаг-3-скрипты-eslint-и-тесты-локально)
7. [Шаг 4. Скрипт сборки](#шаг-4-скрипт-сборки)
8. [Шаг 5. Первый workflow: Checkout → Lint → Test → Build → Upload](#шаг-5-первый-workflow-checkout--lint--test--build--upload)
9. [Шаг 6. Красный → зелёный: проверяем, что CI работает](#шаг-6-красный--зелёный-проверяем-что-ci-работает)
10. [Шаг 7. Артефакты, значок, матрица](#шаг-7-артефакты-значок-матрица)
11. [Шаг 8. Блокировка merge](#шаг-8-блокировка-merge)

**Часть Б. API и контрактное тестирование**

12. [Шаг 9. Контракт `openapi.yaml`](#шаг-9-контракт-openapiyaml)
13. [Шаг 10. Проверка и просмотр спецификации](#шаг-10-проверка-и-просмотр-спецификации)
14. [Шаг 11. Мок-сервер для фронтенда](#шаг-11-мок-сервер-для-фронтенда)
15. [Шаг 12. Контрактные тесты](#шаг-12-контрактные-тесты)
16. [Шаг 13. Ломаем контракт: эксперименты](#шаг-13-ломаем-контракт-эксперименты)
17. [Шаг 14. Контракт в CI](#шаг-14-контракт-в-ci)
18. [Шаг 15. Pull Request с нарушением контракта](#шаг-15-pull-request-с-нарушением-контракта)
19. [Шаг 16. Дополнительные задания](#шаг-16-дополнительные-задания)

20. [Содержание отчёта](#содержание-отчёта)
21. [Критерии оценивания](#критерии-оценивания)
22. [Контрольные вопросы](#контрольные-вопросы)
23. [Приложения](#приложения)

---

## 1. Цель и результаты

**Цель:** научиться автоматизировать сборку и проверку проекта в CI, описывать API формальным контрактом OpenAPI, запускать мок-сервер и писать автотесты, проверяющие соответствие реализации контракту.

**После выполнения работы вы:**

- настроите GitHub Actions: Checkout → Lint → Test → Build → Upload Artifact;
- убедитесь, что пайплайн запускается на каждый Push и Pull Request;
- сделаете проверки обязательными для слияния;
- опишете REST API в файле `openapi.yaml` и проверите его линтером Spectral;
- запустите мок-сервер по контракту и «поработаете как frontend»;
- напишете контрактные тесты и добавите их в CI;
- убедитесь, что нарушение контракта делает пайплайн красным и блокирует merge.

**Итоговые артефакты:**

| Артефакт | Файл / место |
|---|---|
| Workflow CI | `.github/workflows/ci.yml` |
| Контракт API | `openapi.yaml` |
| Правила линтера контракта | `.spectral.yaml` |
| Автотесты контрактов | `tests/contract/rooms.contract.test.js` |
| Зелёный пайплайн с артефактами | вкладка Actions |
| Блокировка merge | настройки защиты ветки |
| Отчёт | `docs/report.md` |

---

## 2. Подготовка

### 2.1. Что нужно

- аккаунт GitHub;
- Git, Node.js (LTS, v20+), npm; проверьте:

```bash
git --version
node --version
npm --version
```

- редактор кода (желательно VS Code с расширением для OpenAPI/YAML);
- (необязательно) Docker для запуска Swagger UI.

### 2.2. Настройте Git (если ещё не делали)

```bash
git config --global user.name  "Фамилия Имя"
git config --global user.email "ваш_email_с_GitHub"
git config --global init.defaultBranch main
```

### 2.3. Правила работы

- Рабочая ветка — `main` защищена (шаг 8): после этого **все изменения только через Pull Request**.
- Названия коммитов — по Conventional Commits (`feat:`, `fix:`, `ci:`, `docs:`, `test:`).
- Все терминальные команды выполняются **из корня проекта**.

---

## 3. Легенда проекта

Вы разрабатываете **Room Booking API** — небольшой REST-сервис учёта аудиторий университета. В работе реализуются четыре операции:

| Метод и путь | Назначение |
|---|---|
| `GET /rooms` | Список аудиторий (необязательный фильтр `minCapacity`) |
| `POST /rooms` | Создание аудитории |
| `GET /rooms/{roomId}` | Получение аудитории по идентификатору |
| `DELETE /rooms/{roomId}` | Удаление аудитории |

Сервис хранит данные **в памяти** (без базы): нас интересует процесс, а не хранение.

```
Сценарий работы:
  Frontend-разработчик  ──(мок по openapi.yaml)──►  разрабатывает интерфейс
  Backend-разработчик   ──(реализует тот же контракт)──► Express-приложение
  CI                    ──► lint → test (в т.ч. контрактные) → build → artifact
```

---

# Часть А. CI-пайплайн

## Шаг 1. Репозиторий

### 1.1. Создайте репозиторий

1. На GitHub: **«+» → New repository**.
2. Имя: `contract-ci-lab`.
3. Тип: **Public** (защита веток на бесплатном тарифе для приватных репозиториев может быть недоступна).
4. Отметьте **Add a README file**, в **.gitignore** выберите шаблон **Node**.
5. **Create repository**.

### 1.2. Настройте слияние

**Settings → General → Pull Requests:**

- Allow squash merging — ✅ (единственный способ);
- Allow merge commits и rebase merging — ❌;
- Automatically delete head branches — ✅.

### 1.3. Клонируйте

```bash
git clone https://github.com/<логин>/contract-ci-lab.git
cd contract-ci-lab
```

Если вы работаете в паре, добавьте напарника: **Settings → Collaborators → Add people**.

**Проверьте:** репозиторий создан и склонирован, доступ у напарника есть.

---

## Шаг 2. Учебное приложение

Приложение — небольшое, но с честной логикой: валидация, коды ошибок, хранилище. Создайте файлы.

### 2.1. Инициализация проекта

```bash
npm init -y
npm install express
npm install --save-dev jest supertest eslint @eslint/js globals
```

### 2.2. Файл `src/store.js` — хранилище

```js
'use strict';

class RoomStore {
  constructor(seed = []) {
    this.rooms = new Map();
    this.nextId = 1;
    seed.forEach((room) => this.create(room));
  }

  list(minCapacity) {
    const all = [...this.rooms.values()];
    if (minCapacity === undefined) {
      return all;
    }
    return all.filter((room) => room.capacity >= minCapacity);
  }

  get(id) {
    return this.rooms.get(id);
  }

  findByName(name) {
    return [...this.rooms.values()].find((room) => room.name === name);
  }

  create({ name, capacity, equipment = [] }) {
    const room = { id: this.nextId, name, capacity, equipment };
    this.rooms.set(room.id, room);
    this.nextId += 1;
    return room;
  }

  delete(id) {
    return this.rooms.delete(id);
  }
}

module.exports = { RoomStore };
```

### 2.3. Файл `src/validation.js` — проверка входных данных

```js
'use strict';

const MAX_NAME_LENGTH = 100;
const MAX_CAPACITY = 1000;

function validateRoomInput(body) {
  if (typeof body.name !== 'string' || body.name.trim() === '') {
    return 'Field "name" is required and must be a non-empty string';
  }
  if (body.name.length > MAX_NAME_LENGTH) {
    return `Field "name" must be at most ${MAX_NAME_LENGTH} characters`;
  }
  if (!Number.isInteger(body.capacity) || body.capacity < 1 || body.capacity > MAX_CAPACITY) {
    return `Field "capacity" must be an integer between 1 and ${MAX_CAPACITY}`;
  }
  if (body.equipment !== undefined) {
    const isValid =
      Array.isArray(body.equipment) && body.equipment.every((item) => typeof item === 'string');
    if (!isValid) {
      return 'Field "equipment" must be an array of strings';
    }
  }
  return null;
}

function parsePositiveInt(value) {
  if (typeof value !== 'string' || !/^\d+$/.test(value)) {
    return null;
  }
  const number = Number(value);
  return number >= 1 ? number : null;
}

module.exports = { validateRoomInput, parsePositiveInt };
```

### 2.4. Файл `src/app.js` — приложение

```js
'use strict';

const express = require('express');
const { RoomStore } = require('./store');
const { validateRoomInput, parsePositiveInt } = require('./validation');

const SEED_ROOMS = [
  { name: 'A-101', capacity: 30, equipment: ['projector'] },
  { name: 'B-205', capacity: 120, equipment: ['projector', 'microphone'] },
];

function errorBody(code, message) {
  return { code, message };
}

function createApp(store = new RoomStore(SEED_ROOMS)) {
  const app = express();
  app.use(express.json());

  app.get('/rooms', (req, res) => {
    let minCapacity;
    if (req.query.minCapacity !== undefined) {
      minCapacity = parsePositiveInt(req.query.minCapacity);
      if (minCapacity === null) {
        return res
          .status(400)
          .json(errorBody('VALIDATION_ERROR', 'Query "minCapacity" must be a positive integer'));
      }
    }
    return res.status(200).json(store.list(minCapacity));
  });

  app.post('/rooms', (req, res) => {
    const body = req.body ?? {};
    const error = validateRoomInput(body);
    if (error) {
      return res.status(400).json(errorBody('VALIDATION_ERROR', error));
    }
    if (store.findByName(body.name)) {
      return res
        .status(409)
        .json(errorBody('CONFLICT', `Room "${body.name}" already exists`));
    }
    const room = store.create(body);
    return res.status(201).location(`/rooms/${room.id}`).json(room);
  });

  app.get('/rooms/:roomId', (req, res) => {
    const id = parsePositiveInt(req.params.roomId);
    if (id === null) {
      return res
        .status(400)
        .json(errorBody('VALIDATION_ERROR', 'Path "roomId" must be a positive integer'));
    }
    const room = store.get(id);
    if (!room) {
      return res.status(404).json(errorBody('NOT_FOUND', `Room ${id} not found`));
    }
    return res.status(200).json(room);
  });

  app.delete('/rooms/:roomId', (req, res) => {
    const id = parsePositiveInt(req.params.roomId);
    if (id === null) {
      return res
        .status(400)
        .json(errorBody('VALIDATION_ERROR', 'Path "roomId" must be a positive integer'));
    }
    if (!store.delete(id)) {
      return res.status(404).json(errorBody('NOT_FOUND', `Room ${id} not found`));
    }
    return res.status(204).send();
  });

  app.use((req, res) => {
    res.status(404).json(errorBody('NOT_FOUND', 'Route not found'));
  });

  // Обработчик ошибок (например, некорректный JSON в теле запроса)
  app.use((err, req, res, _next) => {
    const status = err.status && err.status < 500 ? err.status : 500;
    const code = status === 400 ? 'VALIDATION_ERROR' : 'INTERNAL_ERROR';
    res.status(status).json(errorBody(code, status === 400 ? 'Malformed request' : 'Internal error'));
  });

  return app;
}

module.exports = { createApp, SEED_ROOMS };
```

### 2.5. Файл `src/server.js` — запуск

```js
'use strict';

const { createApp } = require('./app');

const PORT = process.env.PORT || 3000;

createApp().listen(PORT, () => {
  console.log(`Room Booking API listening on http://localhost:${PORT}`);
});
```

### 2.6. Unit-тесты `tests/unit/store.test.js`

```js
'use strict';

const { RoomStore } = require('../../src/store');
const { validateRoomInput, parsePositiveInt } = require('../../src/validation');

describe('RoomStore', () => {
  test('create присваивает возрастающие идентификаторы', () => {
    const store = new RoomStore();
    const first = store.create({ name: 'A', capacity: 10 });
    const second = store.create({ name: 'B', capacity: 20 });
    expect(first.id).toBe(1);
    expect(second.id).toBe(2);
  });

  test('list фильтрует по вместимости', () => {
    const store = new RoomStore([
      { name: 'small', capacity: 10 },
      { name: 'large', capacity: 100 },
    ]);
    expect(store.list(50)).toHaveLength(1);
    expect(store.list()).toHaveLength(2);
  });

  test('delete возвращает false для несуществующей аудитории', () => {
    const store = new RoomStore();
    expect(store.delete(42)).toBe(false);
  });
});

describe('validation', () => {
  test('validateRoomInput принимает корректные данные', () => {
    expect(validateRoomInput({ name: 'A-1', capacity: 10 })).toBeNull();
  });

  test('validateRoomInput отклоняет пустое имя', () => {
    expect(validateRoomInput({ name: '  ', capacity: 10 })).toMatch(/name/);
  });

  test('validateRoomInput отклоняет вместимость 0', () => {
    expect(validateRoomInput({ name: 'A-1', capacity: 0 })).toMatch(/capacity/);
  });

  test('parsePositiveInt', () => {
    expect(parsePositiveInt('5')).toBe(5);
    expect(parsePositiveInt('0')).toBeNull();
    expect(parsePositiveInt('abc')).toBeNull();
    expect(parsePositiveInt(undefined)).toBeNull();
  });
});
```

**Проверьте (пока без запуска):** создано 5 файлов в `src/` и `tests/unit/`.

---

## Шаг 3. Скрипты, ESLint и тесты локально

### 3.1. Скрипты и настройка Jest в `package.json`

Откройте `package.json` и приведите его к виду (версии зависимостей оставьте те, что установил npm):

```json
{
  "name": "contract-ci-lab",
  "version": "1.0.0",
  "private": true,
  "description": "Учебный проект: CI и контрактное тестирование API",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "lint": "eslint . --max-warnings 0",
    "test": "jest",
    "test:unit": "jest tests/unit",
    "test:ci": "jest --coverage",
    "build": "node scripts/build.js"
  },
  "jest": {
    "testEnvironment": "node",
    "collectCoverageFrom": ["src/**/*.js", "!src/server.js"]
  }
}
```

(Не удаляйте разделы `dependencies` и `devDependencies`, которые добавил npm.)

> Скрипты `test:contract`, `lint:api`, `docs`, `mock` добавим в части Б.

### 3.2. Конфигурация ESLint `eslint.config.js`

```js
'use strict';

const js = require('@eslint/js');
const globals = require('globals');

module.exports = [
  js.configs.recommended,
  {
    languageOptions: {
      ecmaVersion: 2022,
      sourceType: 'commonjs',
      globals: { ...globals.node, ...globals.jest },
    },
    rules: {
      eqeqeq: 'error',
      'no-var': 'error',
      'prefer-const': 'error',
      curly: ['error', 'all'],
      'no-console': 'warn',
      'no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    },
  },
  {
    // В запускаемых файлах и скриптах вывод в консоль допустим
    files: ['src/server.js', 'scripts/**/*.js'],
    rules: { 'no-console': 'off' },
  },
  {
    ignores: ['node_modules/', 'dist/', 'coverage/'],
  },
];
```

### 3.3. Запустите проверки локально

```bash
npm run lint
npm test
```

Ожидаемо: линтер без замечаний, все тесты зелёные. Если `npm run lint` сообщает о проблемах, исправьте их.

### 3.4. Запустите сервер руками

```bash
npm start
```

В другом терминале:

```bash
curl -i http://localhost:3000/rooms
curl -i http://localhost:3000/rooms/1
curl -i http://localhost:3000/rooms/999
curl -i -X POST http://localhost:3000/rooms \
  -H "Content-Type: application/json" \
  -d '{"name":"C-301","capacity":50,"equipment":["whiteboard"]}'
```

Остановите сервер (`Ctrl+C`).

> В Windows PowerShell команда `curl` может быть алиасом на `Invoke-WebRequest`: используйте `curl.exe` либо Git Bash.

**В отчёт:** вывод `npm run lint`, `npm test` и одного запроса `curl`.

---

## Шаг 4. Скрипт сборки

Для приложения на Node.js «сборка» — это сбор готового к поставке набора файлов в каталог `dist/` с информацией о версии. Создайте `scripts/build.js`:

```js
'use strict';

const fs = require('node:fs');
const path = require('node:path');

const root = path.join(__dirname, '..');
const dist = path.join(root, 'dist');
const pkg = require('../package.json');

fs.rmSync(dist, { recursive: true, force: true });
fs.mkdirSync(dist, { recursive: true });

// Что попадает в поставку (отсутствующие файлы пропускаются)
const include = ['src', 'package.json', 'package-lock.json', 'openapi.yaml'];
for (const item of include) {
  const from = path.join(root, item);
  if (fs.existsSync(from)) {
    fs.cpSync(from, path.join(dist, item), { recursive: true });
  }
}

const info = {
  name: pkg.name,
  version: pkg.version,
  commit: process.env.GITHUB_SHA || process.env.CI_COMMIT_SHA || 'local',
  ref: process.env.GITHUB_REF_NAME || process.env.CI_COMMIT_REF_NAME || 'local',
  builtAt: new Date().toISOString(),
};
fs.writeFileSync(path.join(dist, 'build-info.json'), JSON.stringify(info, null, 2));

console.log('Build complete:', info);
```

Проверьте:

```bash
npm run build
ls dist
cat dist/build-info.json
```

Ожидаемо: в `dist/` лежат `src/`, `package.json`, `package-lock.json`, `build-info.json`. Каталог `dist/` уже в `.gitignore` не входит по умолчанию для шаблона Node в некоторых версиях; проверьте и при необходимости добавьте строки `dist/` и `coverage/` в `.gitignore`.

Теперь закоммитьте проект. Это **первый и последний** прямой push в `main` (защиту включим позже):

```bash
git add .
git commit -m "feat: add room booking api with unit tests"
git push origin main
```

**Проверьте:** код на GitHub; `npm run build` создаёт `dist/` с `build-info.json`.

---

## Шаг 5. Первый workflow: Checkout → Lint → Test → Build → Upload

Теперь автоматизируем всё это в CI.

### 5.1. Создайте ветку

```bash
git switch -c ci/add-pipeline
mkdir -p .github/workflows
```

### 5.2. Файл `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  lint:
    name: lint
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: ESLint
        run: npm run lint

  test:
    name: test
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Run tests with coverage
        run: npm run test:ci

      - name: Upload coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7
          if-no-files-found: ignore

  build:
    name: build
    needs: [lint, test]
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-${{ github.sha }}
          path: dist/
          retention-days: 7
          if-no-files-found: error
```

### 5.3. Отправьте и откройте Pull Request

```bash
git add .
git commit -m "ci: add lint, test and build pipeline"
git push -u origin ci/add-pipeline
```

На GitHub нажмите **Compare & pull request**. Заголовок: `ci: add lint, test and build pipeline`. Создайте PR.

### 5.4. Изучите запуск

Откройте вкладку **Checks** (или **Actions**) в PR:

1. Видны три задания: `lint`, `test`, `build`.
2. `lint` и `test` идут **параллельно**, `build` ждёт их.
3. Откройте задание → разверните шаги → найдите в логах `npm ci`, вывод ESLint, таблицу покрытия.
4. После успеха на странице запуска (в блоке **Artifacts**) появятся `coverage-report` и `app-<sha>`.

**В отчёт:**
- скриншот графа заданий (`lint` ∥ `test` → `build`);
- скриншот блока Artifacts;
- ответ: сколько секунд заняло каждое задание и почему `build` не стартовал сразу.

> Если запуск красный, откройте лог упавшего шага и разберитесь в причине (Приложение Б). Частая причина — рассинхронизация `package.json` и `package-lock.json` (исправляется `npm install` и коммитом lock-файла).

### 5.5. Слияние

Пока защита `main` не включена, можно слить PR (**Squash and merge**). Затем:

```bash
git switch main
git pull
git branch -d ci/add-pipeline
```

**Проверьте:** на `main` после слияния запустился ещё один зелёный workflow (событие `push`).

---

## Шаг 6. Красный → зелёный: проверяем, что CI работает

CI, который никогда не краснеет, ничего не проверяет. Намеренно сломаем и починим. Для каждого эксперимента создавайте отдельную ветку и PR.

### 6.1. Эксперимент 1: нарушение линтера

```bash
git switch -c exp/lint-fail
```

В `src/store.js` добавьте строку внутри класса `RoomStore` (например, после метода `get`):

```js
  unusedMethod() {
    var x = 1;
  }
```

```bash
git add .
git commit -m "test: break lint on purpose"
git push -u origin exp/lint-fail
```

Откройте PR. Ожидаемо:

- `lint` — ❌ (ошибки `no-var`, `no-unused-vars`);
- `test` — ✅;
- `build` — **пропущен** (skipped), так как `needs` не выполнен.

### 6.2. Эксперимент 2: падающий тест

В той же ветке верните файл (`git checkout main -- src/store.js`) и сломайте тест: в `tests/unit/store.test.js` замените ожидание `expect(second.id).toBe(2)` на `toBe(3)`.

```bash
git add .
git commit -m "test: break unit test on purpose"
git push
```

Ожидаемо: `lint` ✅, `test` ❌, `build` пропущен. В логе шага «Run tests with coverage» виден diff ожидаемого и фактического значения. Обратите внимание, что отчёт покрытия всё равно выгружен (`if: always()`).

### 6.3. Исправление

Верните код и тесты к состоянию `main` и убедитесь, что пайплайн снова зелёный:

```bash
git checkout main -- src tests
git commit -m "fix: restore code"
git push
```

### 6.4. Заполните таблицу

| Эксперимент | `lint` | `test` | `build` | Объяснение |
|---|:---:|:---:|:---:|---|
| Исходный код | ✅ | ✅ | ✅ | |
| Нарушение линтера | ❌ | ✅ | ⏭ skipped | `build` зависит от `lint` и `test` |
| Падающий тест | ✅ | ❌ | ⏭ skipped | |
| После исправления | ✅ | ✅ | ✅ | |

📝 **В отчёт:** таблица с вашими фактическими результатами и скриншот красного запуска. Закройте PR `exp/lint-fail` без слияния, удалите ветку.

---

## Шаг 7. Артефакты, значок, матрица

### 7.1. Скачайте артефакт

На странице успешного запуска (**Actions → запуск → Artifacts**) скачайте `app-<sha>`. Распакуйте и проверьте содержимое:

```bash
unzip app-*.zip -d downloaded
cat downloaded/build-info.json
```

Убедитесь, что `commit` в `build-info.json` совпадает с хэшем коммита запуска. Это показывает, **из какого именно кода собран артефакт**.

### 7.2. Значок статуса в README

В `README.md` добавьте в начало (подставьте свои значения):

```markdown
![CI](https://github.com/<логин>/contract-ci-lab/actions/workflows/ci.yml/badge.svg)

# Room Booking API

Учебный проект: CI-пайплайн, OpenAPI-контракт и контрактные тесты.
```

### 7.3. Матрица версий Node.js

В job `test` замените блок на матрицу:

```yaml
  test:
    name: test
    runs-on: ubuntu-latest
    timeout-minutes: 10
    strategy:
      fail-fast: false
      matrix:
        node: [20, 22]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: npm
      - run: npm ci
      - name: Run tests with coverage
        run: npm run test:ci
      - name: Upload coverage report
        if: always() && matrix.node == 20
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
          retention-days: 7
          if-no-files-found: ignore
```

Теперь `test` запускается дважды параллельно; отчёт покрытия выгружается только из одной ветки матрицы (иначе имена артефактов конфликтовали бы).

> **Внимание.** Имена проверок в матрице: `test (20)` и `test (22)`. В шаге 8 в обязательных проверках нужно указывать именно такие имена. Если усложнять не хочется, пропустите этот подшаг.

### 7.4. Оформите PR

```bash
git switch -c ci/badge-and-matrix
git add .
git commit -m "ci: add status badge and node version matrix"
git push -u origin ci/badge-and-matrix
```

Откройте PR, дождитесь зелёного результата, слейте.

**Проверьте:** значок в README показывает **passing**; артефакт скачивается и содержит `build-info.json`.

**В отчёт:** скриншот README со значком; содержимое `build-info.json` скачанного артефакта.

---

## Шаг 8. Блокировка merge

### 8.1. Включите защиту `main`

**Settings → Rules → Rulesets → New ruleset → New branch ruleset** (или **Settings → Branches → Add branch protection rule**, шаблон `main`).

| Параметр | Значение |
|---|---|
| Enforcement status | Active |
| Target branches | Default branch (`main`) |
| Require a pull request before merging | ✅ |
| Require status checks to pass | ✅ |
| &nbsp;&nbsp;Required checks | `lint`, `test` (или `test (20)`, `test (22)`), `build` |
| Require branches to be up to date before merging | ✅ |
| Require conversation resolution before merging | ✅ |
| Block force pushes | ✅ |

> Имена проверок появятся в списке выбора **после** первого запуска. Если нужной проверки нет, воспользуйтесь поиском внутри списка. Убедитесь, что в списке нет «устаревших» имён, которых больше не существует: иначе PR будет ждать их вечно.

> Если для вашей пары важно ревью, включите также **Require approvals: 1**. Для индивидуальной работы этот пункт можно пропустить (автор не может одобрить собственный PR).

### 8.2. Проверьте блокировку

Создайте ветку с намеренной ошибкой (как в шаге 6.1) и откройте PR. Должно быть:

- внизу PR: «Required statuses must pass before merging»;
- проверка `lint` ❌;
- кнопка **Merge** недоступна.

Исправьте ошибку, дождитесь зелёного статуса, слейте.

📝 **В отчёт:** скриншот заблокированной кнопки Merge при красном CI и скриншот настроек правила. **Это ключевое доказательство «блокировки merge».**

---

# Часть Б. API и контрактное тестирование

С этого момента работаем **design-first**: сначала контракт, затем проверка реализации контракту.

## Шаг 9. Контракт `openapi.yaml`

```bash
git switch main && git pull
git switch -c feat/openapi-contract
```

Создайте в корне проекта файл `openapi.yaml`.

```yaml
openapi: 3.0.3

info:
  title: Room Booking API
  version: 1.0.0
  description: |
    Учебный REST API для учёта аудиторий университета.
    Контракт является единым источником правды для документации,
    мок-сервера и автоматических тестов.
  contact:
    name: Кафедра информационных технологий
    email: dept@example.edu
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: http://localhost:3000
    description: Локальный сервер разработки
  - url: http://localhost:4010
    description: Мок-сервер (Prism)

tags:
  - name: Rooms
    description: Операции с аудиториями

paths:
  /rooms:
    get:
      tags: [Rooms]
      operationId: listRooms
      summary: Получить список аудиторий
      description: >
        Возвращает все аудитории. Можно отфильтровать по минимальной вместимости.
      parameters:
        - name: minCapacity
          in: query
          required: false
          description: Минимальная вместимость аудитории
          schema:
            type: integer
            minimum: 1
            example: 50
      responses:
        '200':
          description: Список аудиторий (может быть пустым)
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Room'
              examples:
                sample:
                  summary: Две аудитории
                  value:
                    - id: 1
                      name: A-101
                      capacity: 30
                      equipment: [projector]
                    - id: 2
                      name: B-205
                      capacity: 120
                      equipment: [projector, microphone]
        '400':
          $ref: '#/components/responses/BadRequest'
    post:
      tags: [Rooms]
      operationId: createRoom
      summary: Создать аудиторию
      description: Создаёт новую аудиторию. Название должно быть уникальным.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RoomCreate'
            example:
              name: C-301
              capacity: 50
              equipment: [whiteboard]
      responses:
        '201':
          description: Аудитория создана
          headers:
            Location:
              description: Адрес созданной аудитории
              schema:
                type: string
                example: /rooms/3
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Room'
        '400':
          $ref: '#/components/responses/BadRequest'
        '409':
          description: Аудитория с таким названием уже существует
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Error'
              example:
                code: CONFLICT
                message: Room "A-101" already exists

  /rooms/{roomId}:
    parameters:
      - name: roomId
        in: path
        required: true
        description: Идентификатор аудитории
        schema:
          type: integer
          minimum: 1
          example: 1
    get:
      tags: [Rooms]
      operationId: getRoom
      summary: Получить аудиторию по идентификатору
      description: Возвращает одну аудиторию или ошибку, если она не найдена.
      responses:
        '200':
          description: Аудитория найдена
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Room'
        '400':
          $ref: '#/components/responses/BadRequest'
        '404':
          $ref: '#/components/responses/NotFound'
    delete:
      tags: [Rooms]
      operationId: deleteRoom
      summary: Удалить аудиторию
      description: Удаляет аудиторию по идентификатору.
      responses:
        '204':
          description: Аудитория удалена
        '400':
          $ref: '#/components/responses/BadRequest'
        '404':
          $ref: '#/components/responses/NotFound'

components:
  schemas:
    Room:
      type: object
      description: Аудитория
      required: [id, name, capacity, equipment]
      additionalProperties: false
      properties:
        id:
          type: integer
          minimum: 1
          example: 1
        name:
          type: string
          minLength: 1
          maxLength: 100
          example: A-101
        capacity:
          type: integer
          minimum: 1
          maximum: 1000
          example: 30
        equipment:
          type: array
          description: Список оборудования
          items:
            type: string
          example: [projector]

    RoomCreate:
      type: object
      description: Данные для создания аудитории
      required: [name, capacity]
      additionalProperties: false
      properties:
        name:
          type: string
          minLength: 1
          maxLength: 100
          example: C-301
        capacity:
          type: integer
          minimum: 1
          maximum: 1000
          example: 50
        equipment:
          type: array
          items:
            type: string
          example: [whiteboard]

    Error:
      type: object
      description: Единый формат ошибки
      required: [code, message]
      properties:
        code:
          type: string
          enum: [VALIDATION_ERROR, NOT_FOUND, CONFLICT, INTERNAL_ERROR]
          example: NOT_FOUND
        message:
          type: string
          example: Room 99 not found

  responses:
    BadRequest:
      description: Некорректный запрос
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: VALIDATION_ERROR
            message: Field "capacity" must be an integer between 1 and 1000
    NotFound:
      description: Ресурс не найден
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/Error'
          example:
            code: NOT_FOUND
            message: Room 99 not found
```

### Разберите контракт

Ответьте письменно (в отчёт):

1. Какие поля обязательны в схеме `Room`? Чем `Room` отличается от `RoomCreate`?
2. Зачем в `Room` указано `additionalProperties: false`?
3. Почему ответы `BadRequest` и `NotFound` вынесены в `components/responses`?
4. Какие коды ответа описаны для `POST /rooms`?
5. Для чего нужны `operationId`?

> Контракт и реализация должны совпадать. Обратите внимание: приложение возвращает у всех аудиторий поле `equipment` (по умолчанию пустой массив). Поэтому в схеме `Room` это поле обязательно.

**Проверьте:** файл `openapi.yaml` сохранён в корне проекта; YAML корректен (отступы — пробелами, не табами).

---

## Шаг 10. Проверка и просмотр спецификации

### 10.1. Линтер Spectral

Установите и создайте конфигурацию.

```bash
npm install --save-dev @stoplight/spectral-cli
```

Файл `.spectral.yaml`:

```yaml
extends: ["spectral:oas"]

rules:
  # Усиливаем стандартные правила
  operation-operationId: error
  operation-description: warn
  operation-tags: error
  info-contact: warn

  # Собственное правило: operationId в camelCase
  operation-id-camel-case:
    description: operationId должен быть в формате camelCase
    severity: error
    given: "$.paths[*][*].operationId"
    then:
      function: pattern
      functionOptions:
        match: "^[a-z][a-zA-Z0-9]+$"
```

Добавьте скрипт в `package.json`:

```json
"lint:api": "spectral lint openapi.yaml --fail-severity=warn"
```

Запустите:

```bash
npm run lint:api
```

Ожидаемо: нет ошибок. Если Spectral выдал замечания (например, `warn` из-за отсутствующих описаний тегов), исправьте их в `openapi.yaml`: цель — **чистый вывод**.

> Флаг `--fail-severity=warn` означает: падать при warning и выше. Без него пострадают только ошибки (error).

### 10.2. Эксперимент: сломайте контракт

1. Удалите `operationId: listRooms` и запустите `npm run lint:api` → ожидаемо ❌ (`operation-operationId`).
2. Верните; замените `operationId: listRooms` на `list_rooms` → ❌ (`operation-id-camel-case`).
3. Укажите в `$ref` несуществующую схему `#/components/schemas/Roomm` → ❌ (битая ссылка).
4. Верните всё как было.

📝 **В отчёт:** вывод Spectral для двух разных нарушений.

### 10.3. Предпросмотр документации

Выберите любой способ:

**Вариант 1 — онлайн:** откройте https://editor.swagger.io и вставьте содержимое `openapi.yaml`. Слева редактор с подсветкой ошибок, справа документация.

> Не вставляйте в онлайн-редакторы закрытые спецификации реальных проектов. Для учебного контракта это безопасно.

**Вариант 2 — Redocly локально:**

```bash
npx --yes @redocly/cli preview-docs openapi.yaml
```

**Вариант 3 — Swagger UI в Docker:**

```bash
docker run --rm -p 8081:8080 -e SWAGGER_JSON=/spec/openapi.yaml \
  -v "$(pwd)":/spec swaggerapi/swagger-ui
```

Откройте http://localhost:8081.

📝 **В отчёт:** скриншот документации; найдите в ней пример запроса для `POST /rooms` и кнопку «Try it out» (если вариант её поддерживает).

---

## Шаг 11. Мок-сервер для фронтенда

Представьте, что вы **frontend-разработчик**, а backend ещё не готов. Контракт есть — поднимаем мок.

### 11.1. Запуск Prism

Добавьте скрипты в `package.json`:

```json
"mock": "npx --yes @stoplight/prism-cli mock openapi.yaml -p 4010",
"docs": "npx --yes @redocly/cli build-docs openapi.yaml -o dist/api-docs.html"
```

Запустите:

```bash
npm run mock
```

Prism выведет список доступных маршрутов. Оставьте его работать.

### 11.2. Поработайте «клиентом»

В другом терминале:

```bash
# 1. Пример из контракта
curl -i http://localhost:4010/rooms

# 2. Одна аудитория
curl -i http://localhost:4010/rooms/1

# 3. Попросите ответ 404 (заголовок Prefer)
curl -i http://localhost:4010/rooms/1 -H "Prefer: code=404"

# 4. Некорректный запрос: roomId не число
curl -i http://localhost:4010/rooms/abc

# 5. Создание с нарушением контракта: capacity = 0
curl -i -X POST http://localhost:4010/rooms \
  -H "Content-Type: application/json" \
  -d '{"name":"X","capacity":0}'

# 6. Корректное создание
curl -i -X POST http://localhost:4010/rooms \
  -H "Content-Type: application/json" \
  -d '{"name":"C-301","capacity":50}'

# 7. Состояние не сохраняется: создали аудиторию, а в списке её нет
curl -s http://localhost:4010/rooms
```

Запишите наблюдения:

| № | Запрос | Что вернул мок | Откуда взялся ответ |
|---|---|---|---|
| 1 | `GET /rooms` | | пример `sample` из контракта |
| 3 | `Prefer: code=404` | | |
| 4 | `GET /rooms/abc` | | |
| 5 | `POST` с `capacity=0` | | |
| 7 | список после создания | | |

> Пример для строки 5: Prism проверяет **входящий запрос** по контракту и отклоняет нарушение (обычно ответом 422 с описанием проблемы). Для строки 7: мок **не хранит состояние**.

### 11.3. Минимальный клиент для проверки (по желанию)

Создайте `scripts/frontend-demo.js`:

```js
'use strict';

const BASE_URL = process.env.API_URL || 'http://localhost:4010';

async function main() {
  const response = await fetch(`${BASE_URL}/rooms`);
  const rooms = await response.json();
  console.log(`Получено аудиторий: ${rooms.length}`);
  for (const room of rooms) {
    console.log(`- ${room.name}: ${room.capacity} мест`);
  }
}

main().catch((error) => {
  console.error('Ошибка клиента:', error.message);
  process.exit(1);
});
```

```bash
node scripts/frontend-demo.js                         # против мока
npm start &                                           # затем против настоящего сервера:
API_URL=http://localhost:3000 node scripts/frontend-demo.js
```

Один и тот же клиентский код работает и с моком, и с реальным сервером: меняется только адрес. Это и есть смысл «параллельной разработки по контракту».

Остановите мок (`Ctrl+C`) и сервер, если запускали.

📝 **В отчёт:** таблица наблюдений; ответ на вопрос «Какие ограничения у мока вы обнаружили?».

---

## Шаг 12. Контрактные тесты

Теперь проверим, что **настоящий** сервер соответствует контракту.

### 12.1. Установка

```bash
npm install --save-dev jest-openapi
```

Скрипт в `package.json`:

```json
"test:contract": "jest tests/contract"
```

### 12.2. Файл `tests/contract/rooms.contract.test.js`

```js
'use strict';

const path = require('node:path');
const request = require('supertest');
const jestOpenAPI = require('jest-openapi').default;
const { createApp, SEED_ROOMS } = require('../../src/app');
const { RoomStore } = require('../../src/store');

// Загружаем контракт: все проверки toSatisfyApiSpec() идут относительно него
jestOpenAPI(path.join(__dirname, '../../openapi.yaml'));

let app;

beforeEach(() => {
  // Для каждого теста — свежее хранилище, чтобы тесты были независимыми
  app = createApp(new RoomStore(SEED_ROOMS));
});

describe('GET /rooms', () => {
  test('200: список аудиторий соответствует контракту', async () => {
    const res = await request(app).get('/rooms');
    expect(res.status).toBe(200);
    expect(res).toSatisfyApiSpec();
    expect(res.body.length).toBeGreaterThan(0);
  });

  test('200: фильтр minCapacity соответствует контракту', async () => {
    const res = await request(app).get('/rooms').query({ minCapacity: 100 });
    expect(res.status).toBe(200);
    expect(res).toSatisfyApiSpec();
    expect(res.body.every((room) => room.capacity >= 100)).toBe(true);
  });

  test('200: пустой список тоже соответствует контракту', async () => {
    const res = await request(app).get('/rooms').query({ minCapacity: 1000 });
    expect(res.status).toBe(200);
    expect(res).toSatisfyApiSpec();
    expect(res.body).toEqual([]);
  });

  test('400: некорректный minCapacity соответствует контракту ошибки', async () => {
    const res = await request(app).get('/rooms').query({ minCapacity: 'abc' });
    expect(res.status).toBe(400);
    expect(res).toSatisfyApiSpec();
  });
});

describe('POST /rooms', () => {
  test('201: создание аудитории', async () => {
    const res = await request(app)
      .post('/rooms')
      .send({ name: 'C-301', capacity: 50, equipment: ['whiteboard'] });
    expect(res.status).toBe(201);
    expect(res).toSatisfyApiSpec();
    expect(res.headers.location).toBe(`/rooms/${res.body.id}`);
  });

  test('201: equipment необязателен', async () => {
    const res = await request(app).post('/rooms').send({ name: 'D-1', capacity: 10 });
    expect(res.status).toBe(201);
    expect(res).toSatisfyApiSpec();
    expect(res.body.equipment).toEqual([]);
  });

  test('400: нарушена валидация capacity', async () => {
    const res = await request(app).post('/rooms').send({ name: 'Bad', capacity: 0 });
    expect(res.status).toBe(400);
    expect(res).toSatisfyApiSpec();
  });

  test('400: пустое тело запроса', async () => {
    const res = await request(app).post('/rooms').send({});
    expect(res.status).toBe(400);
    expect(res).toSatisfyApiSpec();
  });

  test('409: дублирующееся название', async () => {
    const res = await request(app).post('/rooms').send({ name: 'A-101', capacity: 20 });
    expect(res.status).toBe(409);
    expect(res).toSatisfyApiSpec();
  });
});

describe('GET /rooms/{roomId}', () => {
  test('200: существующая аудитория', async () => {
    const res = await request(app).get('/rooms/1');
    expect(res.status).toBe(200);
    expect(res).toSatisfyApiSpec();
    expect(res.body).toSatisfySchemaInApiSpec('Room');
  });

  test('404: аудитория не найдена', async () => {
    const res = await request(app).get('/rooms/999');
    expect(res.status).toBe(404);
    expect(res).toSatisfyApiSpec();
  });

  test('400: некорректный идентификатор', async () => {
    const res = await request(app).get('/rooms/abc');
    expect(res.status).toBe(400);
    expect(res).toSatisfyApiSpec();
  });
});

describe('DELETE /rooms/{roomId}', () => {
  test('204: аудитория удалена', async () => {
    const res = await request(app).delete('/rooms/1');
    expect(res.status).toBe(204);
    expect(res).toSatisfyApiSpec();
  });

  test('404: повторное удаление', async () => {
    await request(app).delete('/rooms/1');
    const res = await request(app).delete('/rooms/1');
    expect(res.status).toBe(404);
    expect(res).toSatisfyApiSpec();
  });
});
```

### 12.3. Запустите

```bash
npm run test:contract
npm test
```

Ожидаемо: все тесты зелёные.

> **Если тест падает с сообщением вроде «expected res to satisfy API spec»,** прочитайте подробности ошибки: `jest-openapi` показывает, какое именно поле не соответствует схеме или какой код ответа не описан. Это не «ошибка инструмента», а как раз то, ради чего он нужен: либо исправьте реализацию, либо контракт, либо тест.

### 12.4. Что проверяет `toSatisfyApiSpec()`

Выберите правильные утверждения и впишите в отчёт:

- [ ] проверяет, что в контракте есть такой путь и метод;
- [ ] проверяет, что код ответа описан в контракте для этой операции;
- [ ] проверяет, что тело ответа соответствует схеме;
- [ ] проверяет, что сервер выполняет бизнес-правила правильно;
- [ ] проверяет производительность.

*(Правильные — первые три. Бизнес-логика и производительность контрактным тестом не проверяются.)*

📝 **В отчёт:** вывод `npm run test:contract` (количество пройденных тестов).

---

## Шаг 13. Ломаем контракт: эксперименты

Убедимся, что контрактные тесты действительно ловят расхождения. Для каждого эксперимента вносим изменение в **код**, запускаем `npm run test:contract`, фиксируем результат и откатываем изменение (`git checkout -- src/`).

| №   | Изменение                                                                                                               | Ожидаемое поведение теста                           |
| --- | ----------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| 1   | В `GET /rooms/:roomId` вернуть `capacity` строкой: `res.status(200).json({ ...room, capacity: String(room.capacity) })` | тип поля не соответствует схеме                     |
| 2   | В `store.create` добавить в объект лишнее поле `secret: 'x'`                                                            | из-за `additionalProperties: false`                 |
| 3   | Для `POST /rooms` вернуть статус `200` вместо `201`                                                                     | код 200 не описан для `POST`                        |
| 4   | Для «не найдено» вернуть `{ error: 'not found' }` вместо `{ code, message }`                                            | тело не соответствует схеме `Error`                 |
| 5   | Для `DELETE` вернуть `200` с телом `{ ok: true }`                                                                       | код 200 не описан                                   |
| 6   | Удалить из `openapi.yaml` описание `'404'` у `GET /rooms/{roomId}` (код остался прежним)                                | реализация возвращает код, которого нет в контракте |
| 7   | Изменить в `openapi.yaml` `minimum` у `capacity` схемы `Room` на `5`, а в тестах создать аудиторию с `capacity: 1`      | ответ нарушает уточнённую схему                     |

Заполните таблицу:

| № | Что изменили | Упал ли тест | Какой именно | Текст сообщения (кратко) |
|---|---|---|---|---|
| 1 | | | | |
| … | | | | |

**Вывод для отчёта (3–5 предложений):** какие расхождения не смог бы обнаружить unit-тест хранилища? Что пришлось бы делать вручную без контрактных тестов?

> **Дополнительно (эксперимент 8).** Выполните эксперимент 2 и **удалите** `additionalProperties: false` из схемы `Room`. Тест становится зелёным. Что это говорит о строгости контракта? Когда лишние поля допустимы, а когда нет?

---

## Шаг 14. Контракт в CI

Добавим проверки контракта в пайплайн: линт спецификации, контрактные тесты и публикацию документации как артефакта.

### 14.1. Обновите workflow

В `.github/workflows/ci.yml`:

**1) В job `lint` добавьте шаг после `ESLint`:**

```yaml
      - name: Lint OpenAPI contract (Spectral)
        run: npm run lint:api
```

**2) В job `test` добавьте явный шаг для контрактных тестов (после основного прогона):**

```yaml
      - name: Contract tests
        run: npm run test:contract
```

> `npm run test:ci` уже запускает все тесты, включая контрактные. Отдельный шаг нужен для **наглядности**: в интерфейсе GitHub сразу видно, что упали именно контрактные тесты. Если вы не хотите выполнять тесты дважды, измените `test:ci` на `jest --coverage tests/unit` и оставьте контрактные в отдельном шаге.

**3) В job `build` после шага `Build` добавьте генерацию документации:**

```yaml
      - name: Build API documentation
        run: npm run docs
```

Шаг `Upload build artifact` уже выгружает весь каталог `dist/`, поэтому `dist/api-docs.html` попадёт в артефакт вместе со сборкой.

### 14.2. Итоговая структура workflow

```
ci.yml
├── lint     : Checkout → Setup → Install → ESLint → Spectral (openapi.yaml)
├── test     : Checkout → Setup → Install → Unit+Contract (coverage) → Contract tests → Upload coverage
└── build    : (needs lint, test) Checkout → Setup → Install → Build → Build API docs → Upload artifact
```

### 14.3. Отправьте изменения

```bash
git add .
git commit -m "feat(api): add openapi contract, spectral lint and contract tests"
git push -u origin feat/openapi-contract
```

Откройте PR. Дождитесь зелёных проверок `lint`, `test`, `build`. Скачайте артефакт и откройте `dist/api-docs.html` в браузере — документация API должна отобразиться.

✅ **Проверьте:**
- в логе `lint` виден шаг Spectral;
- в логе `test` видны контрактные тесты;
- артефакт содержит `openapi.yaml` и `api-docs.html`.

📝 **В отчёт:** скриншот зелёного пайплайна, содержимое итогового `ci.yml`, скриншот страницы `api-docs.html`.

Слейте PR (Squash and merge, заголовок по Conventional Commits).

---

## Шаг 15. Pull Request с нарушением контракта

Финальная проверка всей цепочки: **случайное изменение API должно быть остановлено автоматически**.

### 15.1. Сценарий

Разработчик решил «чуть улучшить» API: переименовать поле `capacity` в `seats` в ответах — и не обновил контракт.

```bash
git switch main && git pull
git switch -c feat/rename-capacity-to-seats
```

В `src/store.js` в методе `create` замените в создаваемом объекте `capacity` на `seats`:

```js
const room = { id: this.nextId, name, seats: capacity, equipment };
```

(Фильтр в `list` тоже начнёт работать неверно: это нормальное «побочное» последствие.)

```bash
git add .
git commit -m "feat: rename capacity to seats"
git push -u origin feat/rename-capacity-to-seats
```

### 15.2. Ожидаемый результат

В PR:

- `lint` ✅ (код по стилю корректен);
- `test` ❌ (упали как unit-тесты фильтра, так и контрактные: тело не содержит обязательное поле `capacity` и содержит лишнее `seats`);
- `build` — пропущен;
- кнопка Merge заблокирована.

Откройте лог упавшего шага `Contract tests`, найдите сообщение о нарушении схемы.

### 15.3. Два способа выйти из ситуации

Обсудите в паре и выполните **один**:

**А. «Это была ошибка»:** откатите изменение в коде. Пайплайн зелёный.

**Б. «Это осознанное изменение API»:** тогда изменение нужно оформить корректно:
1. Обновить контракт `openapi.yaml` (изменить схемы, примеры).
2. Поскольку это **breaking change**, поднять `info.version` до `2.0.0` (или оставить `capacity` и добавить новое поле `seats`, пометив `capacity` как `deprecated`).
3. Обновить тесты и документацию.
4. Предупредить потребителей (описание PR).

Для отчёта опишите выбранный путь и его обоснование.

### 15.4. Закройте или слейте

Вариант А: закройте PR и удалите ветку. Вариант Б: доведите до зелёного состояния и слейте.

📝 **В отчёт:** скриншот красного PR с заблокированной кнопкой Merge; ссылка на сообщение об ошибке контрактных тестов; выбранный вариант (А/Б) и аргументы.

---

## Шаг 16. Дополнительные задания

*Для желающих (+10 баллов). Достаточно выполнить любые два.*

### 16.1. Новый ресурс `bookings`

Расширьте контракт и реализацию: `POST /bookings` (поля `roomId`, `date`, `startTime`, `endTime`) и `GET /bookings/{id}`. Опишите схемы, коды (201/400/404/409 при пересечении), добавьте примеры, контрактные тесты и мок-сценарии. Выполните **design-first**: сначала PR только с изменением `openapi.yaml`, затем PR с реализацией.

### 16.2. Генерация типов для клиента

```bash
npx --yes openapi-typescript openapi.yaml -o client/api-types.ts
```

Изучите сгенерированный файл. Какие типы появились для `Room` и `Error`? Что произойдёт у клиента (на этапе компиляции TypeScript), если вы переименуете поле в контракте и перегенерируете типы?

### 16.3. Генеративное тестирование (Schemathesis)

```bash
pip install schemathesis
# Запустите сервер: npm start
schemathesis run openapi.yaml --url http://localhost:3000
```

> Синтаксис параметров зависит от версии Schemathesis (`--url` или `--base-url`): сверьтесь со справкой `schemathesis run --help`.

Опишите: какие запросы инструмент сгенерировал, нашёл ли он расхождения (например, `GET /rooms/0`, очень длинное имя, неожиданные типы)? Исправьте найденное либо в коде, либо в контракте.

### 16.4. Проверка breaking changes (oasdiff)

Добавьте в CI job, который сравнивает `openapi.yaml` из PR с версией в `main` и падает при несовместимых изменениях. Пример логики:

```yaml
  contract-compat:
    name: contract-compat
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Get base contract
        run: git show origin/${{ github.base_ref }}:openapi.yaml > /tmp/openapi-base.yaml
      - name: Check for breaking changes
        run: |
          docker run --rm -v "$PWD":/work -v /tmp:/base tufin/oasdiff \
            breaking /base/openapi-base.yaml /work/openapi.yaml --fail-on ERR
```

> Образ, флаги и формат вызова зависят от версии oasdiff; сверьтесь с документацией https://github.com/oasdiff/oasdiff. Протестируйте на PR, который удаляет поле или меняет тип.

### 16.5. Прокси-проверка через Prism

```bash
npm start &
npx --yes @stoplight/prism-cli proxy openapi.yaml http://localhost:3000 --errors
```

Отправляйте запросы через порт прокси и наблюдайте, как Prism сообщает о нарушениях контракта в ответах реального сервера. Сравните этот способ с контрактными тестами.

---

## Содержание отчёта

Отчёт оформляется в Markdown (`docs/report.md` в репозитории) или в PDF.

1. **Титульная информация:** ФИО, группа, состав пары, дата.
2. **Ссылки:** репозиторий, успешный запуск Actions, PR с нарушением контракта.
3. **Часть А:**
   4. Результаты локальных запусков (шаг 3).
   5. Скриншот графа заданий и артефактов (шаг 5), время заданий.
   6. Таблица экспериментов «красный → зелёный» (шаг 6).
   7. Содержимое `build-info.json` из скачанного артефакта, значок в README (шаг 7).
   8. Доказательство блокировки merge (шаг 8).
9. **Часть Б:**
   10. Ответы по контракту (шаг 9).
   11. Вывод Spectral на нарушениях, скриншот документации (шаг 10).
   12. Таблица наблюдений за мок-сервером и вывод об его ограничениях (шаг 11).
   13. Вывод контрактных тестов и ответ про `toSatisfyApiSpec()` (шаг 12).
   14. Таблица экспериментов с нарушением контракта и вывод (шаг 13).
   15. Итоговый `ci.yml`, скриншот зелёного пайплайна, страница `api-docs.html` (шаг 14).
   16. Описание PR с нарушением контракта и выбранное решение (шаг 15).
17. **Дополнительные задания** (если выполнены).
18. **Выводы:** что даёт CI; как контракт связывает frontend и backend; какие ошибки не поймают контрактные тесты; что бы вы добавили в пайплайн в реальном проекте.

---

## Критерии оценивания

| Критерий | Баллы |
|---|---|
| Приложение, скрипты, локальные lint/test/build работают | 8 |
| Workflow CI: этапы Checkout → Lint → Test → Build → Upload, `needs`, кэш | 12 |
| Эксперименты «красный/зелёный» выполнены и объяснены | 8 |
| Артефакты (сборка, покрытие) выгружаются и проверены; значок статуса | 6 |
| Защита `main`: обязательные проверки, доказана блокировка merge | 8 |
| Контракт `openapi.yaml`: полнота (все коды, схемы, примеры, `$ref`) | 14 |
| Spectral настроен, спецификация проходит линтер, собственное правило | 6 |
| Мок-сервер запущен, таблица наблюдений, понимание ограничений | 8 |
| Контрактные тесты: покрытие успешных и ошибочных ответов | 10 |
| Эксперименты с нарушением контракта, выводы | 6 |
| Контракт в CI (Spectral + тесты + артефакт с документацией) | 6 |
| PR с нарушением контракта: красный CI, заблокированный merge, решение | 4 |
| Качество отчёта и выводов | 4 |
| **Итого** | **100** |
| *Дополнительные задания* | *+10* |

**Шкала:** 85–100 — «отлично»; 70–84 — «хорошо»; 50–69 — «удовлетворительно»; менее 50 — «не зачтено».

**Штрафы:** секрет/токен в репозитории — −10; необоснованное отключение проверок (например, удаление теста ради зелёного CI или `eslint-disable` без причины) — до −10; прямые коммиты в `main` после включения защиты — −5.

---

## Контрольные вопросы

1. Из каких job состоит ваш пайплайн и почему `build` зависит от `lint` и `test`?
2. Почему используется `npm ci`, а не `npm install`?
3. Чем артефакт отличается от кэша? Что вы сохраняете как артефакт?
4. Зачем в шаге выгрузки отчёта покрытия стоит `if: always()`?
5. Что нужно сделать, чтобы красный CI блокировал слияние?
6. Чем design-first отличается от code-first? Какой подход вы использовали?
7. Зачем в схеме `Room` указано `additionalProperties: false`? Какие плюсы и минусы?
8. Что такое мок-сервер и чем он отличается от настоящего API?
9. Что проверяет контрактный тест и чего не проверяет?
10. Почему Spectral и контрактные тесты не заменяют друг друга?
11. Какие изменения API считаются breaking? Как их выявить автоматически?
12. Как бы вы организовали контрактное тестирование для десяти микросервисов?

---

## Приложения

### Приложение А. Эквивалент для GitLab CI

Файл `.gitlab-ci.yml` в корне проекта:

```yaml
image: node:20

workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH && $CI_OPEN_MERGE_REQUESTS
      when: never
    - if: $CI_COMMIT_BRANCH

stages:
  - lint
  - test
  - build

cache:
  key:
    files:
      - package-lock.json
  paths:
    - .npm/

variables:
  npm_config_cache: "$CI_PROJECT_DIR/.npm"

before_script:
  - npm ci

lint:
  stage: lint
  script:
    - npm run lint
    - npm run lint:api

test:
  stage: test
  script:
    - npm run test:ci
    - npm run test:contract
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    when: always
    paths:
      - coverage/
    expire_in: 7 days

build:
  stage: build
  script:
    - npm run build
    - npm run docs
  artifacts:
    name: "app-$CI_COMMIT_SHORT_SHA"
    paths:
      - dist/
    expire_in: 7 days
```

Блокировка merge: **Settings → Merge requests → Pipelines must succeed**. Просмотр артефактов: страница pipeline → job → **Browse / Download**. Значок: **Settings → CI/CD → General pipelines → Pipeline status**.

### Приложение Б. Частые проблемы и решения

| Проблема | Причина | Решение |
|---|---|---|
| `npm ci` падает: lock-файл не синхронизирован | `package.json` изменён без обновления `package-lock.json` | `npm install`, закоммитить `package-lock.json` |
| CI не запускается | Файл не в `.github/workflows/` или расширение не `.yml/.yaml`; ошибка отступов | Проверить путь и YAML (пробелы, не табы) |
| `lint` падает локально «работает» | Разные версии пакетов; не закоммичен конфиг | Закоммитить `eslint.config.js`; `npm ci` локально |
| Артефакт не появился | Неверный `path`; каталог не создан | Проверить путь; `if-no-files-found: error` |
| `build` всегда «skipped» | Предыдущие job упали | Починить `lint`/`test` |
| В списке обязательных проверок нет нужной | Workflow ещё не запускался | Запустить на PR, подождать |
| PR вечно «ждёт» проверку | В обязательных старое имя проверки (например, `test` вместо `test (20)`) | Исправить список в правиле |
| `jest-openapi`: `Cannot find module` | Не установлен пакет | `npm install --save-dev jest-openapi` |
| `jest-openapi`: ошибка загрузки спецификации | Ошибка в YAML или неверный путь | Прогнать `npm run lint:api`; проверить путь через `path.join(__dirname, …)` |
| Контрактный тест: «not defined in the spec» | Путь/метод/код ответа отсутствует в `openapi.yaml` | Описать в контракте либо исправить реализацию |
| Spectral: `info-contact`/`operation-description` | Нет описаний | Дописать в `openapi.yaml` |
| Prism: порт занят | Другой процесс на 4010 | Использовать `-p 4011` либо остановить процесс |
| `curl` на Windows работает странно | Алиас PowerShell | `curl.exe` или Git Bash |
| `fetch is not defined` в `frontend-demo.js` | Node.js старше 18 | Обновить Node.js до LTS |
| Ошибка прав при `git push` в `main` | Включена защита | Работать через ветку и PR |

### Приложение В. Итоговая структура репозитория

```
contract-ci-lab/
├── .github/
│   └── workflows/
│       └── ci.yml
├── scripts/
│   ├── build.js
│   └── frontend-demo.js
├── src/
│   ├── app.js
│   ├── server.js
│   ├── store.js
│   └── validation.js
├── tests/
│   ├── contract/
│   │   └── rooms.contract.test.js
│   └── unit/
│       └── store.test.js
├── docs/
│   └── report.md
├── .gitignore
├── .spectral.yaml
├── eslint.config.js
├── openapi.yaml
├── package.json
├── package-lock.json
└── README.md
```

### Приложение Г. Итоговый `package.json` — раздел `scripts`

```json
"scripts": {
  "start": "node src/server.js",
  "lint": "eslint . --max-warnings 0",
  "lint:api": "spectral lint openapi.yaml --fail-severity=warn",
  "test": "jest",
  "test:unit": "jest tests/unit",
  "test:contract": "jest tests/contract",
  "test:ci": "jest --coverage",
  "build": "node scripts/build.js",
  "docs": "npx --yes @redocly/cli build-docs openapi.yaml -o dist/api-docs.html",
  "mock": "npx --yes @stoplight/prism-cli mock openapi.yaml -p 4010"
}
```

### Приложение Д. Шпаргалка по командам

| Задача | Команда |
|---|---|
| Установить зависимости по lock-файлу | `npm ci` |
| Проверить код | `npm run lint` |
| Проверить контракт | `npm run lint:api` |
| Все тесты | `npm test` |
| Только контрактные тесты | `npm run test:contract` |
| Собрать проект | `npm run build` |
| Собрать документацию API | `npm run docs` (после `build`) |
| Запустить мок | `npm run mock` |
| Запустить сервер | `npm start` |
| Запустить workflow вручную | вкладка Actions → CI → Run workflow |
