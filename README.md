# 📚 LibraTrack — Система обліку бібліотеки / Library Management System

**Go with GenAI, Lesson 10 — Full Stack.** Вебзастосунок на Go, Gin,
`html/template` і HTMX із PostgreSQL та GORM. База: Lesson 9, commit `ea8ccb6`.
Збережено REST API, Swagger, bcrypt/JWT, RBAC і транзакційний облік видач.
Додано каталог, браузерний вхід, CRUD книг, живий пошук і вбудовані ресурси.

## 🛠️ Швидкий запуск

Потрібні Go **1.27.2**, Docker Desktop із Compose v2 та Python 3.
З кореня проєкту:

```bash
python3 scripts/setup_env.py
docker compose up -d --wait postgres
go run ./cmd/demo-users
go run ./cmd/api
```

- Каталог: <http://localhost:8085/catalog>.
- Swagger: <http://localhost:8085/swagger/index.html>.
- JSON API: <http://localhost:8085/api/v1/books>.
- PostgreSQL: `127.0.0.1:5440`; окремий volume та бази `libratrack10`,
  `libratrack10_test`, `libratrack10_migr`.

Setup створює випадкові секрети в ігнорованому `.env` із правами `0600`.
`demo-users` створює читача, бібліотекаря й адміністратора та зберігає їхні
випадкові паролі у приватному `.demo-accounts.txt` (`0600`, поза Git).
Повторний запуск зберігає наявний файл і паролі. Реєстрація через браузер
створює лише `reader`; роль із запиту користувача не приймається.

Наявні `.env`, бази та volumes попередніх уроків не використовуються.
Сервер за замовчуванням слухає loopback. Для контейнера задайте
`LISTEN_HOST=0.0.0.0`. Для HTTPS задайте `COOKIE_SECURE=true`.
Змінні середовища мають пріоритет над `.env`; JWT-секрет обов’язковий.

## 📈 Lesson 10 Report — Вебшар, HTMX і єдиний бінарник

### 1. Архітектура та шаблони

```text
cmd/api/main.go                 запуск сервера
internal/app/app.go             реєстрація API та вебмаршрутів
internal/auth/credentials.go    спільна перевірка email/password
internal/handler/web/           HTML-обробники, cookies, CSRF, інтеграційні тести
internal/repository/            PostgreSQL/GORM, пошук і підрахунок примірників
web/templates.go               ParseFS і окремий набір шаблонів на сторінку
web/static.go                  embed.FS, fs.Sub і роздача локальних ресурсів
web/templates/                 layout, pages, partials
web/static/                    CSS, HTMX і невеликий скрипт обробки помилок
migrations/embed.go            вбудовані versioned SQL-міграції
```

HTML і JSON використовують спільні репозиторії та перевірку облікових даних.
Назви книг, описи й інші дані екрануються через `html/template`.
Набори layout/partials/page розбираються один раз на старті. Рендеринг
буферизований: помилка шаблону не надсилає частково успішну сторінку.
Автори й категорії завантажуються через Preload; доступність агрегується в
основному SQL-запиті. Для запиту даних книг перевірено незмінні **4 SELECT** для 10 і 15 книг;
повна сторінка додатково завантажує категорії та, за наявності сесії, користувача.

Збережено міграції 1–5: початкова схема → нормалізація → auth fields →
демонстраційні дані → видачі. Нова SQL-міграція для вебшару не потрібна.

### 2. Можливості й доступ

| Маршрут | Поведінка / доступ |
|---|---|
| `GET /catalog` | Публічний каталог, фільтр категорії, пошук, до 20 книг на сторінку |
| `GET /books/:id` | Книга, автори, опис і наявність примірників |
| `GET /web/books?q=Go` | Лише HTML-рядки таблиці для живого пошуку |
| `GET /books/new`, `/books/:id/edit` | Форми для librarian/admin |
| `POST /web/books`, `/web/books/:id` | Створення/редагування, валідація, redirect після успіху |
| `DELETE /web/books/:id` | HTMX-видалення для librarian/admin; порожня `200 OK` |
| `GET /books/:id/delete`, `POST /web/books/:id/delete` | Підтвердження й видалення звичайною формою, зокрема без JS |
| `GET/POST /login`, `/register` | Браузерний вхід і реєстрація читача |
| `POST /logout` | Очищення cookies |

Пошук використовує `hx-get`, затримку `300ms`, `hx-target="#book-list"` і
скасування попереднього запиту. Параметризований SQL шукає назву або ISBN;
`%` і `_` у введенні трактуються буквально. Для повної пагінації результатів
є звичайна кнопка «Пошук». Видалення використовує `closest tr`, `outerHTML`
і підтвердження. За помилки рядок зберігається, повідомлення показується окремо.
Книгу з історією видач захищають зовнішні ключі.

### 3. Безпека та ручна частина

Браузерний JWT зберігається у cookie з `HttpOnly`, `SameSite=Lax`, `Path=/` і
строком 24 години. `Secure` вмикається для HTTPS. `localStorage` не
використовується: JavaScript, зокрема XSS, може прочитати з нього токен.
HttpOnly обмежує читання cookie, проте не замінює захисту від XSS та CSRF.

Усі вебоперації зміни, включно з login/logout/register, перевіряють CSRF-токен
із підписаним випадковим nonce, прив’язаний до JWT-сесії. Поточна роль
перевіряється в БД. Гість і reader отримують 403 на керованих вебмаршрутах;
API зберігає розрізнення 401/403. Logout очищує cookies, але не відкликає
раніше скопійований JWT. Сувора CSP дозволяє ресурси лише цього сервера.

[design_v10.txt](design_v10.txt) описує рішення та результати автоматичного
аудиту. **19.10.2026 студент особисто підтвердив ручний аудит Task 1.1:**
переглянуто всі HTML-шаблони й Go-рендерери, використання `html/template`
та відсутність небезпечних перетворень у `template.HTML`; зауважень немає.
Студент зробив власні скриншоти CRUD, HTMX, XSS-перевірки й тестів ізоляції.
[prompts.txt](prompts.txt)
містить специфікації Tasks 2.1–2.2 та інтеграції/перевірок.

### 4. Один бінарник і перевірка незалежності

```bash
CGO_ENABLED=0 go build -o bin/libratrack ./cmd/api
python3 scripts/verify_isolation.py --keep-container
```

Скрипт окремо збирає native і Linux/amd64 бінарники. Native перевіряється
в тимчасовій теці лише з виконуваним файлом. Linux працює в `scratch` як
non-root, з read-only filesystem і без mounts. Перевіряються `/catalog`,
CSS, локальний HTMX, app.js та пошуковий HTML-фрагмент; статика зіставляється
побайтово. Фактичний звіт: [evidence/isolation.json](evidence/isolation.json).

- Збережений контейнер: <http://localhost:8086/catalog>.
- Повторний запуск native із порожньої теки: `python3 scripts/run_isolated.py`,
  після чого <http://localhost:8087/catalog>.
- Зупинка тестового контейнера: `docker stop libratrack10-isolated`.

Вбудовано HTML, CSS, JavaScript і SQL. PostgreSQL та конфігурація/секрети
залишаються зовнішніми; бінарник має відповідати ОС та архітектурі.
`MIGRATIONS_PATH` за замовчуванням порожній і означає вбудовані міграції.
Для розробки можна явно вказати зовнішній шлях. Секрети не вбудовуються.
Звичайний multi-stage образ збирається через `docker build -t libratrack10 .`.

### 5. Перевірки

```bash
go mod tidy
go build ./...
go vet ./...
go test -race -count=1 -json ./... > test-results.jsonl
python3 scripts/test_summary.py test-results.jsonl
python3 scripts/verify_isolation.py --keep-container
```

Тести використовують `libratrack10_test` і випадкову окрему схему для кожної
перевірки. Вони охоплюють успадкований API/RBAC/міграції та нові сценарії:
повний HTML, HTMX-фрагменти, CRUD, 403, cookies, CSRF, XSS, пошук, доступність
і вбудовані міграції з порожньої робочої теки. Пропуски через відсутню БД
не вважаються повною перевіркою. Результати поточного запуску наведено в
[evidence/verification.txt](evidence/verification.txt).

---

# 📚 LibraTrack — Library Management System

**Lesson 10: Full Stack.** Builds on Lesson 9 (`ea8ccb6`) with server-rendered
HTML, reusable layouts/partials, browser authentication, book CRUD, live HTMX
search/deletion and embedded runtime assets. The existing REST API and Swagger
remain available. Gin handlers reuse PostgreSQL/GORM repositories and shared
credential validation.

## 🛠️ Quick Start

Use Go **1.27.2**, Docker Desktop/Compose v2 and Python 3. From the project root:

```bash
python3 scripts/setup_env.py
docker compose up -d --wait postgres
go run ./cmd/demo-users
go run ./cmd/api
```

Open <http://localhost:8085/catalog>. Demo credentials are randomly generated
in the ignored private `.demo-accounts.txt`; `.env` is generated separately.
PostgreSQL uses localhost port **5440** and dedicated Lesson 10 databases/volume.
Browser registration always assigns reader. Set `COOKIE_SECURE=true` for HTTPS.

## 📈 Lesson 10 Report

1. **Templates and architecture:** separate page template sets, shared layout and
   row partials, `html/template` autoescaping, startup parsing and buffered
   rendering. The books query with availability preserves four SELECTs for 10 and 15 books;
   the full page also loads categories and the current user when signed in.
2. **Interactivity:** 300 ms live title/ISBN search returns rows only; staff
   deletion returns empty 200 after successful deletion. Errors preserve the row.
   Ordinary forms support create/edit/delete without JavaScript. Search pagination
   is available through the normal Search submit action.
3. **Security:** JWT in HttpOnly/SameSite cookies, configurable Secure, signed
   session-bound CSRF, current database identity/role checks, local scripts and
   restrictive CSP. Web management denial is 403; API keeps its existing 401/403.
   Logout clears cookies without revoking already copied tokens.
4. **Single binary:** HTML, CSS, HTMX, app.js and SQL migrations use embed.FS.
   Native empty-directory and Linux scratch/non-root/read-only/no-mount checks
   are reproducible with `python3 scripts/verify_isolation.py --keep-container`.
   PostgreSQL and environment secrets remain external.
5. **Validation:** inherited and new real-router/PostgreSQL tests, race detector,
   migration round trips, RBAC, XSS, cookies and CSRF. Actual results are recorded
   in `evidence/verification.txt` and `evidence/isolation.json`.
6. **Deliverables:** `prompts.txt`, `design_v10.txt`, source and local verification
   evidence. On 19 October 2026, the student personally confirmed the Task 1.1
   template/renderer audit without findings and captured the execution screenshots.
