# Практическая работа. Настройка Trunk-Based воркфлоу, pre-commit hooks и шаблона Pull Request

**Дисциплина:** «Технология разработки программного обеспечения»
**Тема:** Настройка Git-воркфлоу и Pre-commit hooks
**Формат:** работа в парах или тройках
**Время:** 4 академических часа (2 пары)
**Инструменты:** Git, Node.js (LTS), npm, GitHub, Husky, lint-staged, ESLint, Prettier, commitlint, GitHub Actions

---

## Содержание

1. [Цель и результаты](#1-цель-и-результаты)
2. [Подготовка](#2-подготовка)
3. [Шаг 1. Репозиторий и настройки слияния](#шаг-1-репозиторий-и-настройки-слияния)
4. [Шаг 2. Каркас проекта](#шаг-2-каркас-проекта)
5. [Шаг 3. ESLint и Prettier](#шаг-3-eslint-и-prettier)
6. [Шаг 4. Husky и lint-staged: хук pre-commit](#шаг-4-husky-и-lint-staged-хук-pre-commit)
7. [Шаг 5. commitlint: хук commit-msg](#шаг-5-commitlint-хук-commit-msg)
8. [Шаг 6. Хук pre-push](#шаг-6-хук-pre-push)
9. [Шаг 7. Проверка хуков: тест «на поломку»](#шаг-7-проверка-хуков-тест-на-поломку)
10. [Шаг 8. Шаблон Pull Request и CODEOWNERS](#шаг-8-шаблон-pull-request-и-codeowners)
11. [Шаг 9. CI в GitHub Actions](#шаг-9-ci-в-github-actions)
12. [Шаг 10. Защита ветки main](#шаг-10-защита-ветки-main)
13. [Шаг 11. Рабочий цикл Trunk-Based: два PR с перекрёстным ревью](#шаг-11-рабочий-цикл-trunk-based-два-pr-с-перекрёстным-ревью)
14. [Шаг 12. Feature flag](#шаг-12-feature-flag)
15. [Дополнительная часть: pre-commit для Python](#дополнительная-часть-pre-commit-для-python)
16. [Содержание отчёта](#содержание-отчёта)
17. [Критерии оценивания](#критерии-оценивания)
18. [Контрольные вопросы](#контрольные-вопросы)
19. [Приложения](#приложения)

---

## 1. Цель и результаты

**Цель:** научиться настраивать воркфлоу командной работы с Git на основе Trunk-Based Development, автоматизировать локальные проверки качества кода и оформлять Pull Request по единому стандарту.

**После выполнения работы вы:**

- настроите репозиторий под Trunk-Based воркфлоу (squash-слияние, защита `main`, автоудаление веток);
- подключите **Husky** и **lint-staged**, чтобы линтер и форматтер запускались до коммита;
- настроите **commitlint** для проверки сообщений коммитов по Conventional Commits;
- создадите **шаблон Pull Request с чек-листом для ревью** и файл `CODEOWNERS`;
- настроите **CI**, дублирующий проверки хуков;
- пройдёте полный цикл: ветка → коммиты → PR → перекрёстное ревью → squash-слияние.

**Итоговые артефакты (что должно быть в репозитории):**

| Артефакт | Файл / место |
|---|---|
| Работающие хуки | `.husky/pre-commit`, `.husky/commit-msg`, `.husky/pre-push` |
| Конфигурация проверок | `eslint.config.js`, `.prettierrc`, `commitlint.config.js`, секция `lint-staged` в `package.json` |
| Шаблон PR с чек-листом | `.github/pull_request_template.md` |
| Автоназначение ревьюеров | `.github/CODEOWNERS` |
| CI | `.github/workflows/ci.yml` |
| История | Минимум 2 слитых Pull Request с ревью |

---

## 2. Подготовка

### 2.1. Что должно быть установлено

Проверьте в терминале:

```bash
git --version      # любая актуальная версия
node --version     # v20 и выше (LTS)
npm --version
```

Если Node.js нет, установите версию **LTS** с https://nodejs.org.

### 2.2. Настройте Git (если ещё не делали)

```bash
git config --global user.name  "Фамилия Имя"
git config --global user.email "ваш_email_с_GitHub"
git config --global init.defaultBranch main
git config --global pull.rebase true
```

> Последняя настройка (`pull.rebase true`) делает `git pull` перебазированием, а не созданием лишних merge-коммитов. Это удобно для линейной истории в Trunk-Based воркфлоу.

### 2.3. Распределите роли в паре

Назовём участников **Студент A** и **Студент B**. Роли симметричные: в шаге 11 вы **меняетесь ролями** автора и ревьюера.

| Участник | Основная ответственность на шагах 1–10 |
|---|---|
| **A** | Создаёт репозиторий, выполняет шаги 1–7 на своей машине |
| **B** | Следит по инструкции, выполняет шаги 8–10 в своей ветке или на следующих этапах после клонирования репозитория (см. пометки) |

Если вы работаете в тройке, третий участник — второй ревьюер, а также ведёт протокол (фиксирует вывод команд и скриншоты).

---

## Шаг 1. Репозиторий и настройки слияния

### 1.1. Создайте репозиторий (Студент A)

1. На https://github.com нажмите **«+» → New repository**.
2. Имя: `tbd-hooks-lab`.
3. Тип: **Public**.

   > Для **Private**-репозиториев на бесплатном тарифе защита веток может быть недоступна. Для этой работы удобнее открытый репозиторий.
4. Отметьте **Add a README file**.
5. В списке **Add .gitignore** выберите шаблон **Node**.
6. Нажмите **Create repository**.

### 1.2. Пригласите напарника

**Settings → Collaborators → Add people** → добавьте Студента B (и преподавателя). Напарник принимает приглашение (письмо или уведомление в GitHub).

### 1.3. Настройте стратегию слияния

Откройте **Settings → General**, прокрутите до раздела **Pull Requests** и установите:

| Параметр | Значение |
|---|---|
| Allow merge commits | **выключить** |
| Allow squash merging | **включить** (Default commit message — *Pull request title and description* или *title*) |
| Allow rebase merging | **выключить** |
| Automatically delete head branches | **включить** |

Нажмите **Save**, если требуется.

> **Зачем:** squash-слияние превращает все коммиты ветки в один аккуратный коммит в `main`, а автоудаление веток поддерживает «короткоживущие ветки» без ручной уборки.

### 1.4. Клонируйте репозиторий (оба)

```bash
git clone https://github.com/<ваш_логин>/tbd-hooks-lab.git
cd tbd-hooks-lab
```

**Проверьте:**
- оба участника имеют доступ к репозиторию;
- в настройках разрешён только **Squash merging**;
- репозиторий склонирован локально.

**В отчёт:** ссылка на репозиторий; скриншот раздела Pull Requests в настройках.

---

## Шаг 2. Каркас проекта

Работаем в **Студенте A**. (Напарник B на этом шаге ничего не коммитит — иначе возникнут конфликты. Он может читать инструкцию и готовить окружение.)

### 2.1. Инициализируйте npm-проект

```bash
npm init -y
```

Откройте `package.json` и приведите к виду:

```json
{
  "name": "tbd-hooks-lab",
  "version": "1.0.0",
  "description": "Учебный проект: Trunk-Based воркфлоу и pre-commit hooks",
  "type": "module",
  "scripts": {
    "test": "node --test",
    "lint": "eslint .",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  },
  "license": "ISC"
}
```

> Поле `"type": "module"` позволяет писать `import`/`export` вместо `require`.

### 2.2. Создайте учебный код и тест

Файл `src/calc.js`:

```js
export function add(a, b) {
  return a + b;
}

export function divide(a, b) {
  if (b === 0) {
    throw new Error('Division by zero');
  }
  return a / b;
}
```

Файл `src/calc.test.js`:

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { add, divide } from './calc.js';

test('add: складывает два числа', () => {
  assert.equal(add(2, 3), 5);
});

test('divide: делит числа', () => {
  assert.equal(divide(6, 3), 2);
});

test('divide: бросает ошибку при делении на ноль', () => {
  assert.throws(() => divide(1, 0), /Division by zero/);
});
```

Запустите тесты:

```bash
npm test
```

Ожидаемо: все три теста проходят.

### 2.3. Закоммитьте каркас (пока без хуков)

Так как защита `main` ещё не включена, первый коммит можно сделать напрямую в `main`. **Это единственное исключение**: дальше все изменения идут только через Pull Request.

```bash
git add .
git commit -m "chore: init project with calc module and tests"
git push origin main
```

**Проверьте:** `npm test` проходит; файлы появились на GitHub.

---

## Шаг 3. ESLint и Prettier

### 3.1. Установите пакеты

```bash
npm install --save-dev eslint @eslint/js globals prettier
```

### 3.2. Настройте ESLint

Создайте `eslint.config.js` (современный «плоский» формат конфигурации):

```js
import js from '@eslint/js';
import globals from 'globals';

export default [
  js.configs.recommended,
  {
    languageOptions: {
      ecmaVersion: 2022,
      sourceType: 'module',
      globals: { ...globals.node },
    },
    rules: {
      eqeqeq: 'error',
      'no-var': 'error',
      'prefer-const': 'error',
      'no-console': 'warn',
    },
  },
  {
    ignores: ['node_modules/', 'coverage/'],
  },
];
```

### 3.3. Настройте Prettier

Файл `.prettierrc`:

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100
}
```

Файл `.prettierignore`:

```
node_modules
package-lock.json
coverage
```

### 3.4. Проверьте, что инструменты работают

```bash
npm run lint
npm run format:check
```

Если Prettier сообщает о несоответствиях в уже созданных файлах, выполните:

```bash
npm run format
```

Затем снова `npm run lint` и `npm run format:check`; обе команды должны завершиться без ошибок.

### 3.5. Коммит

```bash
git add .
git commit -m "chore: add eslint and prettier"
git push origin main
```

**Проверьте:** `npm run lint` и `npm run format:check` проходят без ошибок.

**В отчёт:** вывод `npm run lint` (скриншот или текст).

---

## Шаг 4. Husky и lint-staged: хук pre-commit

### 4.1. Установите пакеты

```bash
npm install --save-dev husky lint-staged
```

### 4.2. Инициализируйте Husky

```bash
npx husky init
```

Команда:
- создаст папку `.husky/` с файлом `pre-commit`;
- добавит в `package.json` скрипт `"prepare": "husky"`.

Проверьте `package.json`: в `scripts` должна появиться строка `"prepare": "husky"`.

> **Как это работает:** скрипт `prepare` запускается автоматически после каждого `npm install`. Поэтому любой, кто клонирует репозиторий и выполнит `npm install`, получит те же хуки без дополнительных действий.

### 4.3. Настройте lint-staged

В `package.json` добавьте секцию (на верхнем уровне, рядом со `scripts`):

```json
"lint-staged": {
  "*.js": [
    "eslint --fix",
    "prettier --write"
  ],
  "*.{json,md,yml,yaml}": [
    "prettier --write"
  ]
}
```

> **Важно:** порядок команд важен. Сначала `eslint --fix` исправляет то, что можно исправить автоматически, затем `prettier --write` приводит форматирование к стандарту. Если ESLint находит ошибку, которую нельзя исправить автоматически, коммит прервётся.

### 4.4. Привяжите lint-staged к хуку

Откройте файл `.husky/pre-commit`. По умолчанию там написано `npm test`. Замените **всё содержимое** на одну строку:

```sh
npx lint-staged
```

> Хук `pre-commit` должен быть **быстрым**. Полные тесты мы отправим в `pre-push` (шаг 6) и CI.

### 4.5. Первый запуск

Создайте файл `src/hello.js` с «грязным» форматированием:

```js
export function hello(name){
    return   "Hello, "+name
}
```

Добавьте его в индекс и сделайте коммит:

```bash
git add src/hello.js
git commit -m "feat: add hello function"
```

Что должно произойти:

1. Husky запускает `pre-commit` → lint-staged.
2. lint-staged применяет `eslint --fix` и `prettier --write` **только к `src/hello.js`**.
3. Файл автоматически переформатируется, а коммит создаётся.

Посмотрите, как выглядит файл после коммита:

```bash
cat src/hello.js
```

Ожидаемо: кавычки одинарные, точка с запятой, нормальные пробелы.

> Если ESLint ругается (например, на `no-console`), исправьте код вручную и повторите `git add` и `git commit`.

**Проверьте:**
- при коммите в консоли видны сообщения lint-staged;
- файл `hello.js` в коммите уже отформатирован.

**В отчёт:** вывод консоли при коммите (скриншот или текст); содержимое `.husky/pre-commit` и секции `lint-staged`.

---

## Шаг 5. commitlint: хук commit-msg

### 5.1. Установите пакеты

```bash
npm install --save-dev @commitlint/cli @commitlint/config-conventional
```

### 5.2. Создайте конфигурацию

Файл `commitlint.config.js`:

```js
export default {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'subject-case': [0],
    'header-max-length': [2, 'always', 100],
  },
};
```

> Правило `subject-case` отключено, чтобы можно было писать описание и по-русски, и по-английски. Тип коммита (`feat`, `fix` и т. д.) при этом всё равно обязателен.

### 5.3. Создайте хук commit-msg

Создайте файл `.husky/commit-msg` со следующим содержимым:

```sh
npx --no -- commitlint --edit "$1"
```

> `$1` — путь к временному файлу с текстом сообщения коммита; Git передаёт его хуку автоматически.

### 5.4. Проверьте вручную

Убедитесь, что commitlint «понимает» сообщения:

```bash
echo "bad message" | npx commitlint
echo "feat: add commitlint config" | npx commitlint
```

Первая команда должна вывести ошибки (✖), вторая — ничего не выводит и завершается успешно.

### 5.5. Проверьте на реальном коммите

```bash
git add .
git commit -m "добавил commitlint"
```

Ожидаемо: коммит **отклонён** (нет типа). Теперь с правильным сообщением:

```bash
git commit -m "chore: add commitlint with conventional config"
```

Ожидаемо: коммит создан.

**Проверьте:** коммит с сообщением без типа блокируется, с правильным — проходит.

**В отчёт:** вывод при блокировке «плохого» сообщения; содержимое `commitlint.config.js` и `.husky/commit-msg`.

---

## Шаг 6. Хук pre-push

Хук `pre-push` срабатывает перед отправкой на сервер — подходящее место для быстрых тестов.

Создайте файл `.husky/pre-push`:

```sh
npm test
```

Проверьте: сломайте любой тест (например, в `src/calc.test.js` замените ожидаемое значение в `add` с `5` на `6`), закоммитьте (изменение пройдёт `pre-commit`, так как линтер не проверяет логику) и попробуйте выполнить `git push`:

```bash
git add src/calc.test.js
git commit -m "test: temporarily break add test"
git push
```

Ожидаемо: `git push` **прерывается**, тесты упали.

Верните тест в исходное состояние и сохраните это отдельным коммитом:

```bash
git add src/calc.test.js
git commit -m "test: restore add test"
```

> Если хотите убрать неудачный коммит из истории до публикации, можно использовать `git reset --soft HEAD~2` и сделать один корректный коммит; в учебном режиме достаточно варианта выше.

Теперь отправьте изменения. Помните, что на этом этапе защита `main` ещё не включена, поэтому отправка напрямую пока допустима (это **последний** прямой push):

```bash
git push origin main
```

**Проверьте:** при падающих тестах `git push` блокируется, при исправленных — проходит.

 **В отчёт:** вывод консоли при заблокированном push.

---

## Шаг 7. Проверка хуков: тест «на поломку»

Проверим, что хуки действительно защищают, и разберёмся, как их можно обойти. Заполните таблицу в отчёте по результатам экспериментов.

> **Все эксперименты выполняйте на тестовой ветке**, чтобы не засорять `main`: `git switch -c experiment/hooks-check`. Ветку потом удалите (`git switch main && git branch -D experiment/hooks-check`).

| № | Эксперимент | Как выполнить | Ожидаемый результат |
|---|---|---|---|
| 1 | Нарушение правила ESLint | Создать файл `src/bad.js` с кодом `var x = 1; if (x == 1) {}` и закоммитить | Коммит блокируется (`no-var`, `eqeqeq`) |
| 2 | Только форматирование | Файл с кривым форматированием, но корректной логикой | Файл автоформатируется, коммит проходит |
| 3 | Плохое сообщение | `git commit -m "fix"` | Блокируется commitlint |
| 4 | Падающий тест | Сломать тест и сделать `git push` | `pre-push` блокирует |
| 5 | Обход хуков | `git commit --no-verify -m "bad message"` | Коммит **проходит** — хуки обойдены! |
| 6 | Файл не в индексе | Изменить файл, но не делать `git add`, и закоммитить другой файл | lint-staged проверяет **только staged** файлы |

> **Вывод к эксперименту 5.** Локальные хуки можно обойти флагом `--no-verify`. Поэтому те же проверки нужно **продублировать в CI** и защитить `main`, чтобы обход не привёл к попаданию плохого кода в основную ветку. Этим займёмся в шагах 9–10.

После экспериментов:

```bash
git switch main
git branch -D experiment/hooks-check
```

**Проверьте:** таблица заполнена фактическими результатами; вы понимаете, где хуки защищают, а где нет.

**В отчёт:** заполненная таблица экспериментов (фактический результат и краткий вывод по каждому).

---

## Шаг 8. Шаблон Pull Request и CODEOWNERS

Следующие изменения вносятся **через Pull Request** уже потому, что мы сначала пройдём этот путь без защиты, а затем включим защиту `main` (шаг 10). Чтобы не усложнять, шаблоны добавим напрямую в `main` (последний прямой push), после чего включим защиту.

### 8.1. Шаблон Pull Request

Создайте файл `.github/pull_request_template.md`:

```markdown
## Что и зачем
<!-- Кратко: какую проблему решает этот PR. 1–3 предложения. -->

## Тип изменения
- [ ] `feat` — новая функциональность
- [ ] `fix` — исправление ошибки
- [ ] `refactor` — рефакторинг без изменения поведения
- [ ] `docs` — документация
- [ ] `test` — тесты
- [ ] `chore` / `ci` — служебные изменения

## Связанная задача
Closes #<!-- номер Issue -->

## Что изменено
<!-- Список ключевых изменений. -->
-

## Как проверить
<!-- Пошаговая инструкция для ревьюера. -->
1.
2.

## Скриншоты / вывод консоли
<!-- Если применимо. -->

---

## Чек-лист автора
- [ ] Ветка короткоживущая и создана от актуального `main`
- [ ] PR небольшой и решает **одну** задачу
- [ ] Хуки (`pre-commit`, `commit-msg`) прошли **без** `--no-verify`
- [ ] `npm run lint` и `npm run format:check` проходят
- [ ] `npm test` проходит локально
- [ ] Добавлены или обновлены тесты для нового поведения
- [ ] Документация / README обновлены (если нужно)
- [ ] В коде нет секретов, паролей, отладочных `console.log`
- [ ] Незавершённая функциональность скрыта за feature flag
- [ ] Заголовок PR оформлен по Conventional Commits (используется при squash)

## Чек-лист ревьюера
**Корректность**
- [ ] Изменение решает поставленную задачу и соответствует критериям приёмки
- [ ] Обработаны граничные случаи и ошибки (пустые значения, `null`, деление на ноль и т. п.)

**Читаемость**
- [ ] Имена переменных и функций понятны
- [ ] Нет дублирования и «мёртвого» кода
- [ ] Функции небольшие и делают одну вещь

**Тесты**
- [ ] Тесты проверяют поведение, а не детали реализации
- [ ] Покрыты и обычный сценарий, и сценарии с ошибками

**Безопасность и надёжность**
- [ ] Нет секретов и чувствительных данных в коде и истории PR
- [ ] Входные данные проверяются

**Прочее**
- [ ] Нет лишних изменений, не относящихся к задаче
- [ ] CI зелёный
- [ ] Все обсуждения закрыты

---
<!-- Пометки в комментариях: `nit:` — необязательное замечание, `blocker:` — блокирует слияние. -->
```

### 8.2. CODEOWNERS

Файл `.github/CODEOWNERS` назначает ревьюеров автоматически. Подставьте **реальные** логины участников команды:

```
# Все файлы по умолчанию ревьюит команда
*                       @логин_студента_A @логин_студента_B

# Конфигурация процессов — обязательно смотрит кто-то из пары
/.github/               @логин_студента_A @логин_студента_B
/.husky/                @логин_студента_A @логин_студента_B
```

> **Важно:** в пути `/.github/` не должно быть ошибок, а логины — писать с `@`. Если логин указан неверно, GitHub отметит строку в файле как некорректную.

### 8.3. Добавьте файлы (последний прямой push в `main`)

```bash
git add .github
git commit -m "docs: add pull request template and CODEOWNERS"
git push origin main
```

### 8.4. Проверьте шаблон

Создайте тестовую ветку и откройте пробный PR (закроем его без слияния):

```bash
git switch -c docs/check-pr-template
echo "" >> README.md
git add README.md
git commit -m "docs: check pr template"
git push -u origin docs/check-pr-template
```

На GitHub нажмите **Compare & pull request**. Описание должно быть **автоматически заполнено** шаблоном с чек-листами. Закройте PR без слияния (**Close pull request**), удалите ветку.

```bash
git switch main
git branch -D docs/check-pr-template
```

**Проверьте:** при создании PR автоматически подставляется шаблон; в нём есть чек-листы автора и ревьюера.

**В отчёт:** скриншот нового PR с подставленным шаблоном; содержимое `CODEOWNERS`.

---

## Шаг 9. CI в GitHub Actions

Создайте файл `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  checks:
    name: checks
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Check formatting
        run: npm run format:check

      - name: Tests
        run: npm test

      - name: Check commit messages (only for PR)
        if: github.event_name == 'pull_request'
        run: npx commitlint --from ${{ github.event.pull_request.base.sha }} --to ${{ github.event.pull_request.head.sha }} --verbose
```

> **Что тут важно:**
> - те же проверки, что и в хуках (линтер, форматирование, сообщения коммитов), плюс **полные тесты**;
> - шаг `npm ci` устанавливает зависимости строго по `package-lock.json`;
> - имя задания `checks` понадобится на шаге 10 как **обязательная проверка**;
> - `fetch-depth: 0` нужен, чтобы commitlint видел все коммиты PR.
>
> Так как мы используем squash-слияние, итоговое сообщение коммита берётся из заголовка PR. Поэтому заголовок PR тоже должен соответствовать Conventional Commits (см. чек-лист автора).

Закоммитьте и отправьте (также последний прямой push):

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add github actions workflow"
git push origin main
```

Откройте вкладку **Actions** в репозитории и убедитесь, что запуск **CI** завершился успешно (зелёная галочка). Если красный крест, откройте лог и найдите, какой шаг упал.

**Проверьте:** на вкладке Actions есть успешный запуск задания `checks`.

**В отчёт:** скриншот успешного запуска CI.

---

## Шаг 10. Защита ветки main

Теперь «запираем» основную ветку: прямые push больше невозможны, всё идёт через PR.

### 10.1. Создайте правило

Вариант с **Rulesets** (современный):

1. **Settings → Rules → Rulesets → New ruleset → New branch ruleset**.
2. **Ruleset Name:** `protect-main`.
3. **Enforcement status:** `Active`.
4. **Target branches → Add target → Include default branch**.

Вариант с классическими правилами: **Settings → Branches → Add branch protection rule**, **Branch name pattern:** `main`.

### 10.2. Включите правила

Отметьте следующие пункты (названия могут немного отличаться):

| Правило | Значение |
|---|---|
| Require a pull request before merging | ✅ |
| &nbsp;&nbsp;Required approvals | **1** |
| &nbsp;&nbsp;Require review from Code Owners | ✅ |
| &nbsp;&nbsp;Dismiss stale approvals when new commits are pushed | ✅ |
| Require conversation resolution before merging | ✅ |
| Require status checks to pass | ✅ |
| &nbsp;&nbsp;Require branches to be up to date before merging | ✅ |
| &nbsp;&nbsp;Status checks that are required | добавить **`checks`** |
| Require linear history | ✅ |
| Block force pushes | ✅ |
| Restrict deletions | ✅ |

> **Если в списке обязательных проверок нет `checks`:** GitHub предлагает только те проверки, которые **уже запускались** в последние дни. Убедитесь, что CI хотя бы раз отработал (шаг 9), и воспользуйтесь поиском в поле выбора.

> **Про «обход» правил.** По умолчанию владелец репозитория может обходить защиту (bypass). В учебной работе **не используйте обход**: правило должно действовать на всех, включая вас. Если в настройках есть пункт вроде **Do not allow bypassing the above settings** (в классических правилах) — включите его.

### 10.3. Проверьте защиту

Попробуйте отправить коммит напрямую в `main`:

```bash
git switch main
echo "test" >> README.md
git add README.md
git commit -m "docs: direct push test"
git push origin main
```

Ожидаемо: **отказ** с сообщением вроде `protected branch hook declined` / `Changes must be made through a pull request`.

Отмените локальный коммит:

```bash
git reset --hard origin/main
```

**Проверьте:** прямой push в `main` невозможен; в настройках видны все перечисленные правила.

**В отчёт:** скриншот настроек правила; вывод консоли с отказом при push в `main`.

---

## Шаг 11. Рабочий цикл Trunk-Based: два PR с перекрёстным ревью

Теперь отработаем настоящий процесс. Каждый участник пары выступает **автором одного PR** и **ревьюером другого**.

### 11.0. Что будем делать

| | PR №1 | PR №2 |
|---|---|---|
| Автор | Студент A | Студент B |
| Ревьюер | Студент B | Студент A |
| Задача | `feat`: функция умножения `multiply` | `feat`: функция вычитания `subtract` |

Допускается заменить задачи другими небольшими, главное — **каждая ветка живёт не более одного занятия**, а PR небольшой.

> Конфликт слияния возникнет **естественно**: обе функции добавляются в один файл `src/calc.js`. Это намеренно: так вы потренируетесь разрешать конфликты в TBD-режиме (см. 11.5).

### 11.1. Подготовка (оба)

Оба участника синхронизируют `main` и подтягивают зависимости (так проверяется, что хуки у напарника установились автоматически):

```bash
git switch main
git pull
npm install
```

Убедитесь, что хуки работают на машине напарника: файл `.husky/pre-commit` существует, а команда `git config core.hooksPath` выводит `.husky/_`.

> Если у напарника хуки не срабатывают: проверьте, что вы выполнили `npm install` (скрипт `prepare` устанавливает Husky), а не просто склонировали репозиторий.

### 11.2. Автор создаёт короткую ветку и реализует задачу

**Студент A:**

```bash
git switch -c feat/multiply
```

Добавьте в `src/calc.js`:

```js
export function multiply(a, b) {
  return a * b;
}
```

Добавьте тесты в `src/calc.test.js`: не забудьте дополнить `import`:

```js
import { add, divide, multiply } from './calc.js';

test('multiply: умножает числа', () => {
  assert.equal(multiply(3, 4), 12);
});

test('multiply: умножение на ноль', () => {
  assert.equal(multiply(5, 0), 0);
});
```

Закоммитьте **небольшими логичными коммитами**:

```bash
git add src/calc.js
git commit -m "feat(calc): add multiply function"

git add src/calc.test.js
git commit -m "test(calc): add tests for multiply"
```

Перед отправкой подтяните свежий `main`:

```bash
git fetch origin
git rebase origin/main
```

Отправьте ветку:

```bash
git push -u origin feat/multiply
```

> `pre-push` запустит тесты. Если они упали, исправьте и отправьте снова.

**Студент B** делает то же самое для `feat/subtract` (функция `subtract(a, b)` и два теста).

### 11.3. Автор открывает Pull Request

1. На GitHub нажмите **Compare & pull request**.
2. **Заголовок** в формате Conventional Commits: `feat(calc): add multiply function`.
3. Заполните шаблон: описание, тип, как проверить, связанная задача (можно создать Issue заранее, `Closes #1`).
4. Пройдитесь по **чек-листу автора** и отметьте выполненное (только честно).
5. Убедитесь, что запросился ревьюер (через `CODEOWNERS`). Если нет — назначьте вручную: **Reviewers** справа.
6. Дождитесь зелёного CI.

### 11.4. Ревью (ревьюер)

Ревьюер открывает PR и:

1. Читает описание и вкладку **Files changed**.
2. Оставляет **минимум 2 осмысленных комментария**: по строкам кода (нажмите «+» рядом с номером строки). Формулируйте вежливо и конкретно. Примеры:
   - `nit: можно переименовать параметры в factor1/factor2 для читаемости`
   - `question: что произойдёт, если передать строку вместо числа?`
   - `blocker: нет теста на отрицательные числа`
3. Проходит по **чек-листу ревьюера** из описания PR и отмечает пункты.
4. Принимает решение: **Review changes → Approve** или **Request changes**.

Автор отвечает на комментарии, при необходимости вносит правки новыми коммитами, нажимает **Resolve conversation** и просит повторное ревью.

> **Хорошее ревью:** критикуем код, а не человека; отделяем блокирующие замечания от необязательных; хвалим удачные решения.

### 11.5. Слияние и разрешение конфликта

1. Первым вливается **PR №1** (Студент A): кнопка **Squash and merge**, заголовок коммита — по Conventional Commits. Ветка удалится автоматически.
2. После этого PR №2 станет «устаревшим»: GitHub потребует **Update branch** или покажет **конфликт** в `src/calc.js`. Это ожидаемо.
3. **Студент B** разрешает ситуацию локально:

```bash
git switch feat/subtract
git fetch origin
git rebase origin/main
```

При конфликте Git остановится. Откройте `src/calc.js`, найдите маркеры `<<<<<<<`, `=======`, `>>>>>>>`, оставьте **обе** функции (`multiply` и `subtract`), удалите маркеры, затем:

```bash
git add src/calc.js
git rebase --continue
git push --force-with-lease
```

> `--force-with-lease` безопаснее обычного `--force`: он не перезапишет чужие изменения в ветке. Используйте его **только** для своей собственной ветки.

4. Дождитесь, чтобы CI прошёл заново; ревьюер подтверждает изменения (если approval сбросился при новых коммитах).
5. Вливаем PR №2 через **Squash and merge**.

### 11.6. Приведите локальные репозитории в порядок (оба)

```bash
git switch main
git pull
git branch -d feat/multiply feat/subtract
git fetch --prune
```

Посмотрите историю:

```bash
git log --oneline --graph -10
```

Ожидаемо: **линейная** история, где каждая функция — один squash-коммит с заголовком по Conventional Commits.

 **Проверьте:**
- слиты 2 PR; у каждого есть ревью с комментариями и approval;
- CI зелёный на обоих;
- ветки удалены, в `main` линейная история;
- конфликт разрешён корректно (обе функции присутствуют).

 **В отчёт:** ссылки на оба PR; скриншот вкладки Files changed с комментариями ревью; вывод `git log --oneline --graph`.

---

## Шаг 12. Feature flag

Продемонстрируем принцип «незавершённый код скрыт за флагом» — основу работы с короткими ветками в TBD.

### 12.1. Сценарий

Нужно добавить функцию `power(a, b)` (возведение в степень), но предполагается, что работа займёт несколько PR. Чтобы не держать долгую ветку, вливаем код за флагом.

### 12.2. Реализация (один из участников, через PR)

Ветка: `feat/power-behind-flag`.

Файл `src/features.js`:

```js
export const FEATURES = {
  power: process.env.FEATURE_POWER === 'true',
};
```

В файле `src/calc.js` добавьте:

```js
import { FEATURES } from './features.js';

export function power(a, b) {
  if (!FEATURES.power) {
    throw new Error('Feature "power" is disabled');
  }
  return a ** b;
}
```

Тесты (`src/power.test.js`):

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';

test('power: по умолчанию выключена', async () => {
  delete process.env.FEATURE_POWER;
  const { power } = await import('./calc.js?default');
  assert.throws(() => power(2, 3), /disabled/);
});

test('power: работает при включённом флаге', async () => {
  process.env.FEATURE_POWER = 'true';
  const { power } = await import('./calc.js?enabled');
  assert.equal(power(2, 3), 8);
});
```

> **Подсказка:** строки вида `./calc.js?default` — приём, который заставляет Node.js заново загрузить модуль, чтобы флаг был прочитан заново. Если у вас получится проще, например, функция `isEnabled(name)`, которая читает окружение при каждом вызове, используйте свой вариант и объясните выбор в отчёте.

### 12.3. Оформите PR и слейте его

Пройдите весь цикл (шаги 11.3–11.5) с перекрёстным ревью. В описании PR явно укажите:

- флаг выключен по умолчанию;
- как его включить для проверки (`FEATURE_POWER=true npm test`);
- план удаления флага после завершения функции.

 **Проверьте:** код влит в `main`, функциональность по умолчанию отключена и проверена тестами в обоих режимах.

 **В отчёт:** ссылка на PR; краткий ответ на вопрос «Почему флаг нужно удалить после включения функции?».

---

## Дополнительная часть: pre-commit для Python

*Для желающих (+10 баллов) или для команд, работающих на Python. Можно выполнить в отдельной ветке или отдельном репозитории.*

### Д.1. Подготовка

```bash
python -m venv .venv
# Linux/macOS:
source .venv/bin/activate
# Windows (PowerShell):
.venv\Scripts\Activate.ps1

pip install pre-commit
```

### Д.2. Конфигурация

Файл `.pre-commit-config.yaml`:

```yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
      - id: detect-private-key

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.5.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format

  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.2.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

> Значения `rev` приведены для примера. Актуальные версии получите командой `pre-commit autoupdate`.

### Д.3. Установка и запуск

```bash
pre-commit install                        # хук pre-commit
pre-commit install --hook-type commit-msg # хук commit-msg
pre-commit run --all-files                # проверить весь проект
```

### Д.4. Задания

1. Создайте файл `app.py` с намеренными проблемами (лишние пробелы в конце строк, неиспользуемый импорт) и закоммитьте; опишите, что исправил `pre-commit` автоматически, а что потребовало ручной правки.
2. Попробуйте закоммитить с сообщением `fix stuff` — проверьте блокировку.
3. Добавьте шаг `pre-commit run --all-files` в CI (`.github/workflows/ci.yml`).
4. Напишите в README инструкцию для новых участников (как установить `pre-commit`).
5. Сравните Husky и `pre-commit` по таблице из лекции (раздел 8.4): что показалось удобнее и почему?

 **В отчёт:** конфигурация, вывод `pre-commit run`, ответы на пункты 1 и 5.

---

## Содержание отчёта

Отчёт оформляется в Markdown (`docs/report.md` в репозитории) или в PDF. Структура:

1. **Титульная информация:** ФИО, группа, роли в паре.
2. **Ссылки:** репозиторий, оба PR из шага 11, PR с feature flag.
3. **Настройки репозитория (шаг 1):** скриншот настроек слияния.
4. **Линтер и форматтер (шаг 3):** конфигурации, результаты запуска.
5. **Хуки (шаги 4–6):** содержимое `.husky/*`, секция `lint-staged`, вывод консоли при срабатывании и при блокировке.
6. **Эксперименты (шаг 7):** заполненная таблица результатов и вывод про `--no-verify`.
7. **Шаблон PR и CODEOWNERS (шаг 8):** текст файлов, скриншот подставленного шаблона.
8. **CI и защита `main` (шаги 9–10):** `ci.yml`, скриншот правил, скриншот отказа при прямом push.
9. **Рабочий цикл (шаг 11):** ссылки на PR, примеры комментариев ревью, описание конфликта и его разрешения, `git log --graph`.
10. **Feature flag (шаг 12):** описание реализации и ответ на вопрос.
11. **Выводы:** какие проблемы возникли, как решали, что даёт каждый элемент (хуки, CI, защита ветки, шаблон PR), что бы вы настроили иначе в реальном проекте.

---

## Критерии оценивания

| Критерий | Баллы |
|---|---|
| Репозиторий настроен: squash-слияние, автоудаление веток, участники | 5 |
| ESLint и Prettier настроены и работают | 10 |
| Husky + lint-staged: `pre-commit` работает (проверено на «грязном» файле) | 15 |
| commitlint: `commit-msg` блокирует неверные сообщения | 10 |
| `pre-push` с тестами работает | 5 |
| Таблица экспериментов (шаг 7) заполнена и осмыслена | 5 |
| Шаблон PR с чек-листами автора и ревьюера, `CODEOWNERS` | 10 |
| CI настроен и проходит | 10 |
| Защита `main` настроена и проверена | 10 |
| Два PR с перекрёстным ревью (содержательные комментарии), корректное разрешение конфликта | 15 |
| Feature flag | 5 |
| Качество отчёта и выводов | 10 |
| **Итого** | **110** |
| *Дополнительная часть (pre-commit для Python)* | *+10* |

Максимум за работу — **100 баллов** (баллы сверх 100 засчитываются как бонус).

**Шкала:** 85–100 — «отлично»; 70–84 — «хорошо»; 50–69 — «удовлетворительно»; менее 50 — «не зачтено».

---

## Контрольные вопросы

1. Какие три хука вы настроили и в какой момент срабатывает каждый?
2. Почему `prepare` в `package.json` делает хуки Husky «общими» для всей команды?
3. Зачем нужен lint-staged, если можно запускать `eslint .` на весь проект?
4. Что произойдёт при `git commit --no-verify` и как команда защищается от такого обхода?
5. Почему тяжёлые тесты не стоит запускать в `pre-commit`?
6. Чем отличается `git push --force` от `git push --force-with-lease`? Когда допустим каждый?
7. Зачем в TBD включают squash-слияние и линейную историю?
8. Какие пункты из чек-листа ревьюера, по вашему мнению, самые важные и почему?
9. Что такое feature flag и какую проблему он решает в Trunk-Based Development?
10. Почему имя задания CI (`checks`) важно для правил защиты ветки?
11. Как бы вы изменили процесс, если бы команда состояла из 10 человек?

---

## Приложения

### Приложение А. Итоговая структура репозитория

```
tbd-hooks-lab/
├── .github/
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   └── workflows/
│       └── ci.yml
├── .husky/
│   ├── commit-msg
│   ├── pre-commit
│   └── pre-push
├── src/
│   ├── calc.js
│   ├── calc.test.js
│   ├── features.js
│   ├── hello.js
│   └── power.test.js
├── .gitignore
├── .prettierignore
├── .prettierrc
├── commitlint.config.js
├── eslint.config.js
├── package.json
├── package-lock.json
└── README.md
```

### Приложение Б. Итоговый `package.json` (ориентир)

```json
{
  "name": "tbd-hooks-lab",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "prepare": "husky",
    "test": "node --test",
    "lint": "eslint .",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  },
  "lint-staged": {
    "*.js": ["eslint --fix", "prettier --write"],
    "*.{json,md,yml,yaml}": ["prettier --write"]
  },
  "devDependencies": {
    "...": "версии установятся автоматически командой npm install"
  }
}
```

### Приложение В. Шпаргалка по командам

| Задача | Команда |
|---|---|
| Создать ветку и перейти | `git switch -c feat/123-name` |
| Подтянуть свежий `main` в ветку | `git fetch origin && git rebase origin/main` |
| Отправить новую ветку | `git push -u origin feat/123-name` |
| Безопасная перезапись своей ветки | `git push --force-with-lease` |
| Отменить последний коммит, сохранив изменения | `git reset --soft HEAD~1` |
| Посмотреть историю графом | `git log --oneline --graph --all` |
| Удалить локальную ветку | `git branch -d feat/123-name` |
| Почистить удалённые ссылки | `git fetch --prune` |
| Проверить сообщение вручную | `echo "feat: text" \| npx commitlint` |
| Запустить lint-staged вручную | `npx lint-staged` |
| Пропустить хуки (**только для экспериментов!**) | `git commit --no-verify` |

### Приложение Г. Частые проблемы и решения

| Проблема                                       | Возможная причина                                 | Решение                                                                                 |
| ---------------------------------------------- | ------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Хук не срабатывает                             | Не выполнен `npm install` (не сработал `prepare`) | Выполните `npm install`; проверьте `git config core.hooksPath` (должен быть `.husky/_`) |
| `husky: command not found`                     | Husky не установлен                               | `npm install --save-dev husky`                                                          |
| Хук не запускается на Linux/macOS              | Нет права на выполнение                           | `chmod +x .husky/pre-commit .husky/commit-msg .husky/pre-push`                          |
| Хуки не находят `node`/`npx` в GUI-клиенте Git | У GUI другая переменная `PATH`                    | Запустите коммит из терминала или настройте PATH в `~/.config/husky/init.sh`            |
| `commitlint` не находит конфигурацию           | Конфиг не в корне или ошибка синтаксиса           | Проверьте имя `commitlint.config.js` и `"type": "module"`                               |
| ESLint: `Cannot find package '@eslint/js'`     | Не установлен пакет                               | `npm install --save-dev @eslint/js globals`                                             |
| Prettier правит `package-lock.json`            | Файл не в `.prettierignore`                       | Добавьте в `.prettierignore`                                                            |
| lint-staged: `No staged files match`           | В индексе нет подходящих файлов                   | Нормально; проверка просто пропускается                                                 |
| Нет `checks` в списке обязательных проверок    | CI ещё не запускался                              | Запустите workflow (шаг 9) и подождите 1–2 минуты                                       |
| Push в `main` проходит, хотя защита включена   | Включён bypass для владельца                      | Отключите обход правил для администраторов                                              |
| PR «не слить»: *Branch is out-of-date*         | Включено «Require branches to be up to date»      | Нажмите **Update branch** или сделайте `rebase` локально                                |
| После `rebase` push отклоняется                | История переписана                                | `git push --force-with-lease` (только в своей ветке!)                                   |
| Шаблон PR не подставляется                     | Неверное имя или расположение файла               | Файл должен называться `pull_request_template.md` и лежать в `.github/` в ветке `main`  |
