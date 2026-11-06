# Практическая работа. Контейнеризация проекта

**Дисциплина:** Разработка программного обеспечения
**Продолжительность:** 4 академических часа
**Форма работы:** индивидуальная

## Цель работы

Получить практические навыки контейнеризации многокомпонентного веб-приложения: написать **многоэтапные (multi-stage) Dockerfile** для бэкенда на ASP.NET Core и фронтенда на Angular, настроить **nginx** для раздачи SPA и проксирования API, описать весь стек в **`docker-compose.yml`**.
## 0. Подготовка рабочего места

### 0.1. Необходимое ПО

| ПО | Версия | Обязательно |
|---|---|---|
| Docker Desktop (Windows/macOS) или Docker Engine + Compose plugin (Linux) | Docker 24+ / Compose v2 | **да** |
| Редактор кода (VS Code, Rider, Visual Studio) | любая | да |
| Git | любая | желательно |
| .NET SDK 10, Node.js 22 | — | **нет** (нужны только для запуска без Docker, п. 1.3) |

### 0.2. Особенности Windows 10/11

1. Включите виртуализацию в BIOS/UEFI (Intel VT-x / AMD-V). Проверка: *Диспетчер задач → Производительность → ЦП → «Виртуализация: Включено»*.
2. Windows 10 должна быть версии **2004 (сборка 19041) или новее**. Установите WSL 2 (PowerShell от администратора):
   ```powershell
   wsl --install
   wsl --set-default-version 2
   ```
3. Установите Docker Desktop, в настройках убедитесь, что включено *Use the WSL 2 based engine*.
4. Docker Desktop должен работать в режиме **Linux containers** (по умолчанию).

### 0.3. Проверка установки

```bash
docker --version
docker compose version
docker run --rm hello-world
```

**Контрольная точка 0.** Команда `hello-world` выводит «Hello from Docker!».

### 0.4. Получение материалов

Распакуйте архив работы. Работайте в каталоге **`starter/`** — там лежат исходники приложений **без** Docker-файлов. Каталог `solution/` содержит эталонное решение — заглядывайте в него только для самопроверки после выполнения шага.

```
docker-lab/
├── 01-Лекция-Контейнеризация-проекта.md
├── 02-Практическая-работа-Контейнеризация.md   ← вы здесь
├── starter/                    ← ваша рабочая папка
│   ├── backend/TodoApi/        ← ASP.NET Core 10 Minimal API + EF Core + PostgreSQL
│   └── frontend/               ← Angular 19 (standalone components)
└── solution/                   ← эталон (Dockerfile, nginx, compose)
```

>  Все команды ниже выполняются из каталога `starter/`, если не сказано иное.

---

## 1. Знакомство с demo-проектом

### 1.1. Бэкенд — `starter/backend/TodoApi`

ASP.NET Core 10 Minimal API для списка задач.

| Метод | Путь | Описание |
|---|---|---|
| GET | `/health` | проверка работоспособности (для healthcheck) |
| GET | `/api/info` | версия .NET, ОС, имя машины (в контейнере — ID контейнера) |
| GET | `/api/todos` | список задач |
| POST | `/api/todos` | создать задачу `{ "title": "..." }` |
| PUT | `/api/todos/{id}` | изменить `{ "title": "...", "isDone": true }` |
| DELETE | `/api/todos/{id}` | удалить |

Изучите `Program.cs` и ответьте (в отчёт):

- откуда берётся строка подключения к БД? (`builder.Configuration.GetConnectionString("Default")`);
- как приложение ведёт себя, если БД ещё не готова? (цикл повторных попыток);
- как создаётся схема БД? (`EnsureCreatedAsync` + начальные данные `SeedAsync`).

### 1.2. Фронтенд — `starter/frontend`

Angular 19, один компонент `AppComponent` и сервис `TodoService`. Обратите внимание: сервис обращается к API по **относительному** пути `/api/todos`, а не по `http://localhost:5000/...`. Это ключевое решение: какой сервер отдал страницу, тот и получит запросы к API. В Docker это будет nginx.

### 1.3. (Необязательно) Запуск без Docker

Если на компьютере есть .NET SDK 10, Node.js 22 и PostgreSQL — запустите проект «по-старому» и зафиксируйте в отчёте, сколько шагов и какое ПО для этого понадобилось:

```bash
# бэкенд (строка подключения — в appsettings.json)
cd backend/TodoApi && dotnet run --urls http://localhost:5000
# фронтенд (во втором терминале; /api проксируется на :5000 через proxy.conf.json)
cd frontend && npm install && npm start      # → http://localhost:4200
```

Именно эту процедуру мы заменим одной командой.

---

## 2. Наивный Dockerfile бэкенда (для сравнения)

Сначала напишем «как не надо», чтобы измерить эффект оптимизации.

**Шаг 2.1.** Создайте файл `backend/Dockerfile.naive`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0
WORKDIR /src
COPY . .
RUN dotnet restore TodoApi/TodoApi.csproj
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
CMD ["dotnet", "run", "--project", "TodoApi/TodoApi.csproj"]
```

**Шаг 2.2.** Соберите образ и засеките время:

```bash
docker build -f backend/Dockerfile.naive -t todo-backend:naive ./backend
```

> На Windows (PowerShell) время можно измерить так:
> `Measure-Command { docker build -f backend/Dockerfile.naive -t todo-backend:naive ./backend }`
> В bash: `time docker build ...`

**Шаг 2.3.** Посмотрите размер:

```bash
docker images todo-backend
```

**Шаг 2.4.** Внесите любое изменение в `Program.cs` (например, поменяйте текст в `/health`) и **пересоберите**. Обратите внимание: шаг `dotnet restore` выполняется **заново**, потому что `COPY . .` инвалидирует кэш.

**В отчёт:** размер образа, время первой сборки, время повторной сборки после изменения кода. Перечислите, какие проблемы этого Dockerfile вы видите (подсказка: их минимум четыре — размер, кэш, режим запуска, пользователь).

---

## 3. Multi-stage Dockerfile бэкенда

**Шаг 3.1.** Создайте `backend/.dockerignore` — исключаем из контекста сборки всё, что не нужно:

```gitignore
**/bin/
**/obj/
**/out/
**/.vs/
**/.vscode/
**/.idea/
**/*.user
.git/
Dockerfile*
.dockerignore
**/.env
```

**Шаг 3.2.** Создайте `backend/Dockerfile`. Пишите его **по этапам**, читая комментарии:

```dockerfile
# syntax=docker/dockerfile:1
ARG DOTNET_VERSION=10.0

# ================= Этап 1: build =================
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
WORKDIR /src

# (а) Только файл проекта → restore. Этот слой кэшируется,
#     пока не изменятся зависимости в .csproj.
COPY TodoApi/TodoApi.csproj TodoApi/
RUN dotnet restore TodoApi/TodoApi.csproj

# (б) Исходники → publish в Release.
COPY TodoApi/ TodoApi/
RUN dotnet publish TodoApi/TodoApi.csproj \
        -c Release \
        -o /app/publish \
        --no-restore \
        /p:UseAppHost=false

# ================= Этап 2: runtime =================
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-alpine AS runtime

# В Alpine нет ICU — ставим для корректной работы культур и Npgsql
RUN apk add --no-cache icu-libs
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false \
    ASPNETCORE_URLS=http://+:8080 \
    ASPNETCORE_ENVIRONMENT=Production

WORKDIR /app
COPY --from=build /app/publish .

# Непривилегированный пользователь из базового образа
USER $APP_UID
EXPOSE 8080

HEALTHCHECK --interval=10s --timeout=3s --start-period=20s --retries=5 \
    CMD wget -qO- http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "TodoApi.dll"]
```

**Шаг 3.3.** Соберите оптимизированный образ:

```bash
docker build -t todo-backend:multistage ./backend
docker images todo-backend
```

**Шаг 3.4.** Измените `Program.cs` и пересоберите. В выводе сборки найдите строку `CACHED` у шага `RUN dotnet restore` — пакеты не скачивались повторно.

**Шаг 3.5.** Изучите слои итогового образа:

```bash
docker history todo-backend:multistage
```

Убедитесь, что в истории **нет** слоёв с SDK и исходниками.

**Шаг 3.6.** Проверьте, от какого пользователя работает процесс:

```bash
docker run --rm --entrypoint id todo-backend:multistage
```

Ожидается `uid=1654(app)` (не `root`).

**Контрольная точка 3.** Образ `todo-backend:multistage` собран, его размер **в разы меньше** наивного (ориентир: ~1 ГБ против ~130 МБ).

📋 **В отчёт:** таблица сравнения (размер, время первой сборки, время пересборки после изменения кода) для `naive` и `multistage`; вывод `docker history`.

---

## 4. Проверка бэкенда с отдельной БД

Прежде чем писать compose, проверим образ вручную — так вы поймёте, что именно автоматизирует Compose.

**Шаг 4.1.** Создайте сеть и запустите PostgreSQL:

```bash
docker network create todo-test
docker run -d --name test-db --network todo-test \
  -e POSTGRES_DB=todo -e POSTGRES_USER=todo -e POSTGRES_PASSWORD=todo \
  postgres:17-alpine
```

**Шаг 4.2.** Запустите бэкенд, передав строку подключения через переменную окружения (обратите внимание на `Host=test-db` — имя контейнера, а не `localhost`):

```bash
docker run -d --name test-api --network todo-test -p 5000:8080 \
  -e "ConnectionStrings__Default=Host=test-db;Database=todo;Username=todo;Password=todo" \
  todo-backend:multistage
```

> В PowerShell перенос строки — обратная кавычка `` ` `` вместо `\`, или пишите команду в одну строку.

**Шаг 4.3.** Проверьте:

```bash
docker logs test-api          # ищем «База данных готова»
curl http://localhost:5000/health
curl http://localhost:5000/api/todos
docker ps                     # через ~20 с в колонке STATUS появится (healthy)
```

> В PowerShell вместо `curl` используйте `curl.exe` или `Invoke-RestMethod http://localhost:5000/api/todos`, либо просто откройте адрес в браузере.

**Шаг 4.4.** Эксперимент: замените в строке подключения `Host=test-db` на `Host=localhost`, перезапустите контейнер и посмотрите логи. Объясните результат в отчёте.

**Шаг 4.5.** Уберите за собой:

```bash
docker rm -f test-api test-db
docker network rm todo-test
```

**Контрольная точка 4.** `/api/todos` возвращает JSON с тремя начальными задачами.

---

## 5. Конфигурация nginx

nginx в нашем стеке:
1. отдаёт собранные файлы Angular;
2. на неизвестные пути отдаёт `index.html` (SPA fallback);
3. проксирует `/api/*` в контейнер `backend`.

**Шаг 5.1.** Создайте каталог `frontend/nginx` и файл `frontend/nginx/default.conf.template`:

```nginx
upstream backend {
    # имя сервиса из docker-compose.yml; подставляется из переменных окружения
    server ${BACKEND_HOST}:${BACKEND_PORT};
}

server {
    listen       80;
    server_name  _;
    root   /usr/share/nginx/html;
    index  index.html;

    # Сжатие
    gzip on;
    gzip_comp_level 5;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;

    # Заголовки безопасности
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;

    # Обратный прокси для API
    location /api/ {
        proxy_pass         http://backend;     # без "/" в конце — путь /api/... сохраняется
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }

    # Health-check самого nginx
    location = /nginx-health {
        access_log off;
        default_type text/plain;
        return 200 "ok\n";
    }

    # Статика с хешами в именах — кэшируем надолго
    location ~* \.(?:js|css|woff2?|ttf|svg|png|jpg|jpeg|gif|ico|webp)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # index.html не кэшируем, чтобы пользователи получали новую версию
    location = /index.html {
        add_header Cache-Control "no-cache";
    }

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

**Почему `.template`?** Официальный образ nginx при старте прогоняет файлы из `/etc/nginx/templates/*.template` через `envsubst` и кладёт результат в `/etc/nginx/conf.d/`. Подставляются только переменные, *заданные в окружении контейнера* (`BACKEND_HOST`, `BACKEND_PORT`), поэтому собственные переменные nginx (`$uri`, `$host`) не затрагиваются.

**В отчёт:** объясните назначение каждого блока `location`. Что изменится, если написать `proxy_pass http://backend/;` (со слешем)?

---

## 6. Multi-stage Dockerfile фронтенда

**Шаг 6.1.** Создайте `frontend/.dockerignore`:

```gitignore
node_modules/
dist/
.angular/
.git/
.vscode/
.idea/
*.log
Dockerfile*
.dockerignore
**/.env
```

>  Если вы запускали `npm install` на Windows (п. 1.3), без этого файла папка `node_modules` (сотни МБ, бинарники под Windows) попадёт в контекст сборки и может сломать сборку в Linux-контейнере.

**Шаг 6.2.** Создайте `frontend/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1

# ================= Этап 1: build =================
FROM node:22-alpine AS build
WORKDIR /app

# (а) Зависимости — отдельным кэшируемым слоем
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm \
    if [ -f package-lock.json ]; then npm ci; else npm install; fi

# (б) Исходники и production-сборка
COPY . .
RUN npx ng build --configuration production

# ================= Этап 2: runtime =================
FROM nginx:1.28-alpine AS runtime

ENV BACKEND_HOST=backend \
    BACKEND_PORT=8080

RUN rm -f /etc/nginx/conf.d/default.conf
COPY nginx/default.conf.template /etc/nginx/templates/default.conf.template

# Angular 17+ кладёт результат в dist/<project>/browser
COPY --from=build /app/dist/todo-web/browser /usr/share/nginx/html

EXPOSE 80
HEALTHCHECK --interval=10s --timeout=3s --retries=3 \
    CMD wget -qO- http://127.0.0.1/nginx-health || exit 1
```

>  В demo-проекте нет `package-lock.json`, поэтому при первой сборке выполнится `npm install`. **Правильная практика** — сгенерировать lock-файл (выполнить `npm install` один раз локально или скопировать его из контейнера, см. задание 10.А) и хранить его в репозитории, чтобы работал воспроизводимый `npm ci`.

**Шаг 6.3.** Соберите образ:

```bash
docker build -t todo-frontend:multistage ./frontend
docker images todo-frontend
```

**Шаг 6.4.** Для сравнения узнайте размер **промежуточного** этапа сборки:

```bash
docker build --target build -t todo-frontend:build-stage ./frontend
docker images todo-frontend
```

**Шаг 6.5.** Загляните внутрь итогового образа:

```bash
docker run --rm todo-frontend:multistage ls -la /usr/share/nginx/html
docker run --rm todo-frontend:multistage sh -c "which node || echo 'node отсутствует'"
```

**Контрольная точка 6.** Итоговый образ фронтенда ~50–60 МБ, промежуточный — более 1 ГБ; Node.js в итоговом образе отсутствует.

> Запустить этот образ отдельно не получится: nginx при старте не найдёт хост `backend` и завершится с ошибкой `host not found in upstream`. Проверьте это (`docker run --rm todo-frontend:multistage`) и объясните в отчёте. Это и решает Compose.

---

## 7. docker-compose.yml для всего стека

**Шаг 7.1.** Создайте в корне `starter/` файл `.env.example` и скопируйте его в `.env`:

```dotenv
POSTGRES_DB=todo
POSTGRES_USER=todo
POSTGRES_PASSWORD=change_me_please
ASPNETCORE_ENVIRONMENT=Production
WEB_PORT=8080
```

```bash
cp .env.example .env            # Windows: copy .env.example .env
```

> `.env` с реальными паролями **не коммитят**: добавьте его в `.gitignore`.

**Шаг 7.2.** Создайте `starter/docker-compose.yml`. Разберите каждый блок:

```yaml
name: todo-lab

services:

  # ---------- База данных ----------
  db:
    image: postgres:17-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-todo}
      POSTGRES_USER: ${POSTGRES_USER:-todo}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-todo_secret}
    volumes:
      - db-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks: [backend-net]

  # ---------- Бэкенд ----------
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    image: todo-lab/backend:1.0
    restart: unless-stopped
    environment:
      ASPNETCORE_ENVIRONMENT: ${ASPNETCORE_ENVIRONMENT:-Production}
      ConnectionStrings__Default: >-
        Host=db;Port=5432;Database=${POSTGRES_DB:-todo};Username=${POSTGRES_USER:-todo};Password=${POSTGRES_PASSWORD:-todo_secret}
    depends_on:
      db:
        condition: service_healthy
    networks: [backend-net, frontend-net]

  # ---------- Фронтенд + nginx ----------
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    image: todo-lab/frontend:1.0
    restart: unless-stopped
    environment:
      BACKEND_HOST: backend
      BACKEND_PORT: "8080"
    ports:
      - "${WEB_PORT:-8080}:80"
    depends_on:
      backend:
        condition: service_healthy
    networks: [frontend-net]

volumes:
  db-data:

networks:
  backend-net:
  frontend-net:
```

Обратите внимание на архитектурные решения:

| Решение | Зачем |
|---|---|
| Порт опубликован **только у `frontend`** | единая точка входа; БД и API недоступны снаружи |
| Две сети | nginx физически не имеет доступа к БД |
| `condition: service_healthy` | бэкенд стартует после готовности БД, nginx — после готовности бэкенда |
| Именованный том `db-data` | данные переживают пересоздание контейнеров |
| `${VAR:-default}` | значения из `.env` с безопасными умолчаниями |
| `$$` в healthcheck | экранирование: переменная раскрывается внутри контейнера, а не Compose |

**Шаг 7.3.** Проверьте синтаксис и итоговую конфигурацию после подстановки переменных:

```bash
docker compose config
```

**Контрольная точка 7.** `docker compose config` выводит конфигурацию без ошибок, переменные из `.env` подставлены.

---

## 8. Запуск «с нуля» и проверка

**Шаг 8.1.** Удалите всё, что было собрано на предыдущих шагах, чтобы проверить запуск действительно «с нуля»:

```bash
docker compose down -v --rmi local
docker builder prune -f
```

**Шаг 8.2.** Главная команда работы:

```bash
docker compose up --build
```

Наблюдайте за логами: сначала `db` становится `healthy`, затем запускается `backend` (строка «База данных готова»), затем `frontend`.

**Шаг 8.3.** Во втором терминале:

```bash
docker compose ps
```

Все три сервиса должны быть в состоянии `running (healthy)`.

**Шаг 8.4.** Откройте <http://localhost:8080>:

- [ ] отображаются три начальные задачи;
- [ ] можно добавить, отметить выполненной и удалить задачу;
- [ ] внизу страницы видна версия .NET и ID контейнера бэкенда;
- [ ] в DevTools браузера (вкладка Network) запросы идут на `http://localhost:8080/api/todos` — **тот же origin**, ошибок CORS нет.

**Шаг 8.5.** Проверьте SPA fallback и прокси:

```bash
curl -i http://localhost:8080/some/spa/route   # 200 и HTML index.html
curl -i http://localhost:8080/api/todos        # 200 и JSON
curl -i http://localhost:8080/nginx-health     # ok
```

**Шаг 8.6.** Убедитесь, что БД и API **недоступны** напрямую:

```bash
curl http://localhost:5432   # ошибка соединения
curl http://localhost:5000   # ошибка соединения
```

**Шаг 8.7.** Проверка сохранности данных. Добавьте 2–3 задачи, затем:

```bash
docker compose down        # контейнеры удалены, том остался
docker compose up -d
```

Задачи на месте. Теперь:

```bash
docker compose down -v     # удалён и том
docker compose up -d
```

Остались только начальные задачи.

**Контрольная точка 8 (итоговая).** Приложение разворачивается одной командой и полностью работает. Сделайте скриншоты страницы и вывода `docker compose ps` для отчёта.

---

## 9. Исследование: кэш, размеры, отказоустойчивость

### 9.1. Сводная таблица размеров

```bash
docker images --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}"
```

Заполните таблицу:

| Образ | Размер | Комментарий |
|---|---|---|
| `todo-backend:naive` | | SDK + исходники |
| `todo-lab/backend:1.0` (multi-stage) | | только runtime + publish |
| `todo-frontend:build-stage` | | Node + node_modules |
| `todo-lab/frontend:1.0` (multi-stage) | | nginx + статика |
| `postgres:17-alpine` | | готовый образ |

Посчитайте, во сколько раз multi-stage уменьшил размер образов.

### 9.2. Кэш при изменении кода

1. Измените заголовок в `frontend/src/app/app.component.html`.
2. Выполните `docker compose up --build -d` и засеките время.
3. Какие шаги сборки выполнились заново, а какие помечены `CACHED`? Пересобирался ли образ бэкенда?

### 9.3. Отказоустойчивость

1. Остановите БД: `docker compose stop db`. Что увидит пользователь на странице? Что в `docker compose logs backend`?
2. Запустите БД обратно: `docker compose start db`. Восстановилась ли работа без перезапуска бэкенда?
3. Завершите главный процесс бэкенда изнутри контейнера: `docker compose exec backend sh -c "kill 1"`. Посмотрите `docker compose ps` через несколько секунд. Перезапустился ли контейнер? Какая настройка за это отвечает? Повторите с `docker compose stop backend` и объясните разницу. (Политика `restart: unless-stopped` перезапускает контейнер, когда процесс завершился сам, но не после ручной остановки командами `stop`/`kill`.)

### 9.4. Работа внутри контейнеров

```bash
docker compose exec db psql -U todo -d todo -c "SELECT * FROM todos;"
docker compose exec frontend cat /etc/nginx/conf.d/default.conf   # конфиг после envsubst
docker compose exec frontend wget -qO- http://backend:8080/health # nginx видит бэкенд
docker compose exec frontend ping -c1 db                           # а БД — нет (другая сеть)
```

 **В отчёт:** результаты всех экспериментов раздела 9 с пояснениями.

---

## 10. Индивидуальные задания

Выполните **два** задания: одно из группы А и одно из группы Б (номер = номер в журнале по модулю количества заданий + 1, или по согласованию с преподавателем).

### Группа А (базовые)

1. **Lock-файл.** Сгенерируйте `package-lock.json` *без установленного на хосте Node.js*, используя контейнер:
   `docker run --rm -v "${PWD}/frontend:/app" -w /app node:22-alpine npm install --package-lock-only`.
   Замените в Dockerfile условную установку на чистый `npm ci`. Сравните время сборки.
2. **Adminer.** Добавьте в compose сервис `adminer` (веб-интерфейс БД) с профилем `tools` так, чтобы он запускался только командой `docker compose --profile tools up`. Подключитесь к БД через него.
3. **Chiseled-образ.** Переведите runtime бэкенда на `mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled`. Сравните размер. Почему перестал работать `HEALTHCHECK` с `wget`, и как перенести проверку здоровья в compose или приложение?
4. **Ограничение ресурсов.** Ограничьте бэкенд 256 МБ памяти и 0,5 CPU (`deploy.resources.limits`). Проверьте через `docker stats`.
5. **Инициализация БД скриптом.** Вместо `SeedAsync` подключите SQL-скрипт через bind mount в `/docker-entrypoint-initdb.d/`. Объясните, почему скрипт выполняется только при первом создании тома.

### Группа Б (продвинутые)

1. **Режим разработки.** Создайте `docker-compose.override.yml` для разработки: бэкенд запускается через `dotnet watch` с bind mount исходников, фронтенд — через `ng serve` с hot reload, порты БД и API опубликованы. Объясните, как Compose объединяет файлы.
2. **Тесты в сборке.** Добавьте в решение проект xUnit-тестов и отдельный этап `test` в Dockerfile бэкенда; сборка образа должна падать, если тесты не проходят. Используйте `docker build --target test`.
3. **Миграции EF Core.** Замените `EnsureCreated` на миграции. Реализуйте применение миграций отдельным одноразовым сервисом `migrator` в compose (`depends_on` с `condition: service_completed_successfully`).
4. **Конфигурация фронтенда во время выполнения.** Сделайте так, чтобы заголовок приложения задавался переменной окружения контейнера `frontend` без пересборки образа (подсказка: генерация `config.json` в скрипте из `/docker-entrypoint.d/`).
5. **Масштабирование.** Запустите 3 экземпляра бэкенда (`docker compose up --scale backend=3`). Убедитесь, что nginx распределяет запросы (поле «Контейнер» внизу страницы меняется). Что нужно изменить в конфигурации nginx, чтобы он обнаруживал новые экземпляры без перезапуска? (директива `resolver 127.0.0.11`).
6. **Сканирование уязвимостей.** Проверьте все образы через `docker scout cves` или Trivy. Сравните количество уязвимостей у образов на Debian, Alpine и chiseled.

---

## Приложение А. Решение проблем

| Проблема                                           | Решение                                                                                                                                    |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `error during connect ... docker_engine` (Windows) | запустите Docker Desktop, дождитесь статуса «Engine running»                                                                               |
| WSL 2: `Virtualization is not enabled`             | включите VT-x/AMD-V в BIOS; компонент «Платформа виртуальной машины»                                                                       |
| `port is already allocated`                        | порт 8080 занят — измените `WEB_PORT` в `.env`                                                                                             |
| На странице «Welcome to nginx!»                    | в Dockerfile неверный путь: нужен `dist/todo-web/browser`                                                                                  |
| `502 Bad Gateway` на `/api`                        | `docker compose logs backend`; бэкенд не запустился или неверный `BACKEND_PORT`                                                            |
| `host not found in upstream "backend"`             | frontend и backend не в общей сети, или backend не запущен                                                                                 |
| Бэкенд: `password authentication failed`           | пароль в `.env` изменён после создания тома → `docker compose down -v`                                                                     |
| Бэкенд: `Connection refused` с `localhost`         | в строке подключения должен быть `Host=db`                                                                                                 |
| `npm ERR!` при сборке фронтенда                    | проверьте `.dockerignore` (нет ли `node_modules` в контексте), повторите с `--no-cache`                                                    |
| Долгая первая сборка                               | нормально: скачиваются SDK (~900 МБ), Node, пакеты. Повторные сборки используют кэш                                                        |
| Изменения кода не применились                      | используйте `docker compose up --build` (без `--build` берётся старый образ)                                                               |
| Нет доступа к Docker Hub / MCR (сеть, лимиты)      | используйте зеркало реестра (`registry-mirrors` в настройках Docker Engine) или заранее загруженные образы (`docker save` / `docker load`) |
