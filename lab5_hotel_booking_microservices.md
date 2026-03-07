# Лабораторная работа №5

## Тема

Реализация архитектуры на основе сервисов (микросервисной архитектуры)

## Цель работы

Получить опыт работы организации взаимодействия сервисов с
использованием контейнеров Docker.

------------------------------------------------------------------------

# Отчёт по работе: Контейнеризация и CI/CD системы бронирования отелей

## Описание проекта

**Система бронирования отелей** --- программная система, позволяющая
пользователям бронировать номера, администраторам управлять гостиницей,
а персоналу выполнять задачи по обслуживанию номеров.

Типы пользователей системы:

-   **Гость** --- бронирует номера через мобильное приложение.
-   **Администратор** --- управляет системой через веб-интерфейс.
-   **Персонал** --- получает задания по обслуживанию номеров.

Система реализована в виде **микросервисной архитектуры**, где каждый
компонент запускается в отдельном Docker‑контейнере.

Используемые технологии:

-   Frontend: React
-   Mobile: React Native
-   Backend: Java + Spring Boot
-   База данных: PostgreSQL
-   Кэш: Redis
-   Контейнеризация: Docker
-   Оркестрация: Docker Compose

------------------------------------------------------------------------

# Пункт 1 --- Контейнеризация: выделение минимум 3 контейнеров

## Архитектура контейнеров

Система разбита на несколько контейнеров, взаимодействующих между собой
через REST API.

Основные контейнеры:

1.  Web приложение (React)
2.  Backend API (Spring Boot)
3.  PostgreSQL база данных
4.  Redis кэш
5.  Внешние сервисы (email, smart‑locks)

```{=html}
```
    Users
     │
     ├─ Guest
     ├─ Admin
     └─ Staff
            │
            ▼
         Web App (React)
            │ REST API
            ▼
        Backend API
     (Java / Spring Boot)
            │
     ┌──────┼───────────┐
     ▼      ▼           ▼
    Postgres Redis   External API
    Database Cache   Locks/Email

## Dockerfile backend

``` dockerfile
FROM openjdk:21-jdk-slim
WORKDIR /app
COPY target/hotel-booking.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

## Dockerfile web

``` dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm","start"]
```

## docker-compose.yml

``` yaml
version: "3.9"

services:

  backend:
    build: ./backend
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

  web:
    build: ./web
    ports:
      - "3000:3000"

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: hotel
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7

volumes:
  postgres_data:
```

## Запуск контейнеров

``` bash
docker compose up --build
```

После запуска:

  | Адрес | Описание |
|------|----------|
| http://localhost:3000 | веб интерфейс |
| http://localhost:8080 | backend API |

------------------------------------------------------------------------

# Пункт 2 --- Непрерывная интеграция (CI)

Для автоматической сборки используется **GitHub Actions**.

CI pipeline выполняет:

1.  сборку backend приложения
2.  запуск тестов
3.  сборку docker образов

Файл конфигурации:

    .github/workflows/ci.yml

## Пример CI pipeline

``` yaml
name: CI

on:
  push:
    branches: [ main, develop ]

jobs:

  build:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - name: Build backend
        run: mvn package

      - name: Run tests
        run: mvn test

      - name: Build Docker image
        run: docker build -t hotel-backend ./backend
```

CI автоматически запускается при каждом `git push`.

------------------------------------------------------------------------

# Пункт 3 --- Интеграционные тесты

Интеграционные тесты проверяют взаимодействие компонентов системы.

Используемые инструменты:

-   JUnit
-   Spring Boot Test
-   MockMvc

Тестируются:

-   получение списка номеров
-   создание бронирования
-   авторизация пользователя

## Пример теста

``` java
@SpringBootTest
@AutoConfigureMockMvc
class BookingControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    void getRooms() throws Exception {
        mockMvc.perform(get("/api/rooms"))
                .andExpect(status().isOk());
    }
}
```

Тесты автоматически запускаются в CI pipeline.

------------------------------------------------------------------------

# Повышенная сложность --- Непрерывное развертывание (CD)

Для автоматического развертывания используется **Docker Hub**.

CD pipeline выполняет:

1.  сборку docker образов
2.  публикацию образов на Docker Hub

Файл:

    .github/workflows/cd.yml

## Пример CD pipeline

``` yaml
name: CD

on:
  push:
    branches: [ main ]

jobs:

  publish:

    runs-on: ubuntu-latest

    steps:

      - uses: actions/checkout@v4

      - name: Login DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and Push
        run: |
          docker build -t username/hotel-backend ./backend
          docker push username/hotel-backend
```

------------------------------------------------------------------------

# Итог

В ходе лабораторной работы была реализована микросервисная архитектура
системы бронирования отелей.

Были выполнены следующие задачи:

-   контейнеризация приложения с использованием Docker
-   настройка взаимодействия сервисов через Docker Compose
-   реализация CI pipeline
-   разработка интеграционных тестов
-   настройка CD и публикация Docker образов

Контейнеризация позволила упростить развертывание системы и
автоматизировать процессы сборки и тестирования.
