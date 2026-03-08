
# Лабораторная работа №5  
## Реализация контейнерной архитектуры и CI/CD для веб‑приложения

---

# Тема
Контейнеризация веб‑приложения и организация CI/CD пайплайна с использованием Docker и GitHub Actions.

---

# Цель работы
Получить практический опыт:
- контейнеризации приложения с помощью Docker
- оркестрации контейнеров через Docker Compose
- настройки непрерывной интеграции (CI)
- настройки непрерывного развёртывания (CD)
- публикации Docker‑образов в Docker Hub
- развёртывания приложения на удалённом VPS сервере

---

# Описание проекта

В рамках лабораторной работы было разработано простое веб‑приложение **Hotel Booking System**.

Приложение позволяет пользователю загрузить список доступных номеров отеля через кнопку **Load rooms**.

Архитектура приложения состоит из двух основных компонентов:

- **Frontend** — статическая веб‑страница (HTML + JavaScript)
- **Backend** — REST API на Node.js, который возвращает данные о номерах

Frontend отправляет HTTP‑запрос к backend сервису и отображает полученные данные.

Для повышения гибкости и масштабируемости приложение было разделено на несколько контейнеров.

---

# Пункт 1 — Контейнеризация приложения

## Архитектура контейнеров

Приложение состоит из следующих контейнеров:

1. **Web container**
   - содержит frontend
   - раздаёт HTML/JS через Nginx

2. **Backend container**
   - Node.js сервер
   - предоставляет API `/rooms`

3. **Database container**
   - PostgreSQL
   - используется для хранения данных приложения

Общая архитектура:

```
Browser
   │
   ▼
Web container (Nginx)
   │
   ▼
Backend container (Node.js API)
   │
   ▼
PostgreSQL database
```

Контейнеры взаимодействуют через внутреннюю Docker‑сеть.

---

# Backend сервис

Backend реализован на **Node.js + Express**.

Он предоставляет REST API.

## Пример API

```
GET /rooms
```

Ответ сервера:

```json
[
  {
    "id": 1,
    "name": "Standard room",
    "price": 100
  },
  {
    "id": 2,
    "name": "Deluxe room",
    "price": 200
  }
]
```

---

# Dockerfile backend

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 8080

CMD ["node", "server.js"]
```

Описание:

- используется лёгкий образ **node:20-alpine**
- копируются зависимости
- выполняется установка npm пакетов
- запускается сервер `server.js`

---

# Dockerfile frontend

Frontend представляет собой статические файлы.

Для их раздачи используется **Nginx**.

```dockerfile
FROM nginx:alpine

COPY . /usr/share/nginx/html
```

Nginx используется как лёгкий HTTP сервер.

---

# Docker Compose

Для управления контейнерами используется файл `docker-compose.yml`.

Он описывает все сервисы приложения.

```yaml
services:

  backend:
    build: ./backend
    ports:
      - "8080:8080"

  web:
    build: ./web
    ports:
      - "3000:80"
    depends_on:
      - backend

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: hotel
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
```

---

# Запуск приложения

Контейнеры запускаются командой:

```
docker compose up --build
```

После запуска доступны следующие адреса:

| Адрес | Описание |
|------|----------|
| http://localhost:3000 | веб интерфейс |
| http://localhost:8080/rooms | API backend |

---

# Пункт 2 — Непрерывная интеграция (CI)

Для автоматизации сборки и тестирования используется **GitHub Actions**.

CI pipeline запускается автоматически при каждом `push` в репозиторий.

Основные этапы пайплайна:

1. клонирование репозитория
2. установка зависимостей backend
3. запуск тестов
4. сборка Docker образов

---

# Файл CI pipeline

`.github/workflows/ci.yml`

```yaml
name: CI

on: [push]

jobs:

  build-and-test:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - name: Install backend dependencies
        working-directory: ./backend
        run: npm install

      - name: Run tests
        working-directory: ./backend
        run: npm test

      - name: Build backend image
        run: docker build -t hotel-backend ./backend

      - name: Build web image
        run: docker build -t hotel-web ./web
```

CI pipeline автоматически проверяет работоспособность проекта при каждом изменении кода.

---

# Пункт 3 — Интеграционные тесты

Для тестирования backend используются:

- **Supertest**

Тесты проверяют корректность работы API.

---

# Пример теста

`backend/tests/test.js`

```javascript
const request = require("supertest");
const app = require("../app/app");

describe("API test", () => {

  it("GET /rooms", async () => {

    const res = await request(app).get("/rooms");

    expect(res.statusCode).toBe(200);

  });

});
```

Тест выполняет реальный HTTP‑запрос к серверу и проверяет статус ответа.

---

# Пункт 4 — Непрерывное развёртывание (CD)

Для автоматической публикации Docker образов используется **CD pipeline**.

Pipeline выполняет:

1. авторизацию в Docker Hub
2. сборку Docker образов
3. публикацию образов

---

# CD workflow

`.github/workflows/cd.yml`

```yaml
- name: Log in to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}

- name: Build backend image
  run: docker build -t ${{ secrets.DOCKER_USERNAME }}/hotel-backend:latest ./backend

- name: Push backend image
  run: docker push ${{ secrets.DOCKER_USERNAME }}/hotel-backend:latest

- name: Build web image
  run: docker build -t ${{ secrets.DOCKER_USERNAME }}/hotel-web:latest ./web

- name: Push web image
  run: docker push ${{ secrets.DOCKER_USERNAME }}/hotel-web:latest
```

После выполнения pipeline Docker‑образы публикуются в **Docker Hub**.

---

# Развёртывание приложения на VPS

Приложение было развёрнуто на удалённом **VPS сервере**.

На сервере были выполнены следующие шаги:

1. установка Docker
2. загрузка Docker образов
3. запуск контейнеров

---

# Загрузка образов

```
docker pull USERNAME/hotel-backend:latest
docker pull USERNAME/hotel-web:latest
```

---

# Запуск контейнеров

```
docker run -d -p 8080:8080 USERNAME/hotel-backend:latest
docker run -d -p 3000:80 USERNAME/hotel-web:latest
```

После запуска приложение стало доступно через публичный IP сервера.

---

# Проверка работы приложения

Frontend:

```
http://SERVER_IP:3000
```

Backend API:

```
http://SERVER_IP:8080/rooms
```

После нажатия кнопки **Load rooms** frontend отправляет запрос к backend и отображает список номеров.

---

# Итоговая архитектура системы

```
Developer
   │
   ▼
GitHub repository
   │
   ▼
GitHub Actions (CI)
   │
   ├── запуск тестов
   └── сборка Docker образов
   │
   ▼
Docker Hub
   │
   ▼
VPS сервер
   │
   ├── docker pull образов
   └── docker run контейнеров
```

---

# Структура проекта

```
hotel-booking
│
├── backend
│   ├── app
│   ├── tests
│   ├── server.js
│   ├── package.json
│   └── Dockerfile
│
├── web
│   ├── index.html
│   └── Dockerfile
│
├── docker-compose.yml
│
└── .github
    └── workflows
        ├── ci.yml
        └── cd.yml
```

---

# Вывод

В ходе лабораторной работы были получены практические навыки:

- контейнеризации приложения с помощью Docker
- оркестрации контейнеров через Docker Compose
- настройки CI pipeline в GitHub Actions
- написания интеграционных тестов
- публикации Docker образов в Docker Hub
- развёртывания контейнеризированного приложения на VPS сервере

Использование контейнеров значительно упрощает развёртывание и масштабирование приложений, а CI/CD автоматизирует процесс сборки, тестирования.
