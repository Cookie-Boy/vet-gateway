# 🚪 Vet Gateway

![Java](https://img.shields.io/badge/Java-21-orange?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.x-6DB33F?logo=springboot&logoColor=white)
![Spring Cloud Gateway](https://img.shields.io/badge/Spring%20Cloud%20Gateway-Reactive-6DB33F?logo=spring&logoColor=white)
![WebFlux](https://img.shields.io/badge/WebFlux-Reactive%20Stack-blueviolet)
![Keycloak](https://img.shields.io/badge/Keycloak-OAuth2-4D4D4D?logo=keycloak&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)

> Единая точка входа в микросервисную платформу Veterinary Platform: маршрутизация, CORS, централизованная аутентификация и логирование.

---

## 📋 Назначение

**Vet Gateway** — входной узел всей платформы, построенный на **Spring Cloud Gateway** (реактивный стек на WebFlux). Он выполняет роль обратного прокси и обеспечивает:

- **Маршрутизацию** — направляет входящий трафик к нужному микросервису по правилам `Path=` с возможностью перезаписи пути (`StripPrefix`, `RewritePath`).
- **Централизованную аутентификацию** — валидирует JWT-токены, выданные Keycloak, до передачи запроса в целевой сервис.
- **CORS-политику** — единую для всех сервисов, с поддержкой credential-запросов с фронтенда.
- **Логирование** — глобальный фильтр фиксирует заголовки всех входящих запросов для трассировки и отладки.
- **Публичные маршруты** — отдельные пути (например, QR-скан питомца) доступны без авторизации.

Таким образом, сам микросервис не хранит бизнес-логику, а служит «фасадом», за которым скрыта вся внутренняя топология платформы.

---

## ✨ Ключевые возможности

### Реактивный API Gateway

Построен на **Spring WebFlux** и **Spring Cloud Gateway** — неблокирующая обработка тысяч одновременных соединений без роста числа потоков. Идеально подходит для IoT-сценариев платформы, где через MQTT-шлюз и REST проходит поток телеметрии.

### Централизованная безопасность

- Все внешние запросы валидируются по JWT, подписанному Keycloak.
- Ресурсный сервер (`oauth2ResourceServer`) настраивается через `issuer-uri` и `jwk-set-uri`, что обеспечивает автоматическую ротацию ключей.
- Публичные маршруты явно перечислены (`OPTIONS` для CORS preflight, `/profile/pets/qr-scan/**` для экстренного доступа к карточке питомца по QR).
- CSRF отключён, поскольку API работает в stateless-режиме.

### CORS для фронтенда

`CorsConfig` регистрирует `CorsWebFilter` для всех путей (`/**`), разрешает любой HTTP-метод и заголовок, а также передачу учётных данных. Допустимый origin задаётся переменной окружения `FRONTEND_URL` — при смене окружения править код не нужно.

### Единое логирование

`LoggingGlobalFilter` — глобальный фильтр, который логирует заголовки всех запросов. Это критично для отладки: видно, пришёл ли JWT, какие есть кастомные заголовки, кто реально обращается к системе.

### Маршрутизация с префиксами

Внешние пути построены единообразно: `/api/<service>/**` со снятием префикса (`StripPrefix=1`). Внутри каждый сервис ничего не знает про свой публичный префикс — это полностью ответственность Gateway. Такой подход упрощает добавление новых сервисов и версионирование API.

---

## 🛠 Технологический стек

| Компонент | Технология |
|---|---|
| Язык | Java 21 |
| Фреймворк | Spring Boot 3, Spring WebFlux |
| Шлюз | Spring Cloud Gateway |
| Безопасность | Spring Security OAuth2 Resource Server, Keycloak |
| Логирование | SLF4J + Logback |
| Конфигурация | `application.properties` + переменные окружения |
| Контейнеризация | Docker (multi-stage build) |

---

## 🏗 Архитектура маршрутизации

```mermaid
flowchart LR
    WEB["🌐 Web App"] -->|"JWT"| GW["🚪 Vet Gateway :8888"]
    VK["💬 VK Bot"] -.-> GW

    GW -->|"/auth/**"| KC["🔐 Keycloak :8080"]
    GW -->|"/api/appointment/**"| AS["📅 Appointment Service :8081"]
    GW -->|"/api/management/**"| MS["🏥 Management Service :8082"]
    GW -->|"/api/profile/**"| PS["👤 Profile Service :8083"]
    GW -->|"/api/health/**"| HA["🧠 Health Analyzer :8084"]
    GW -->|"/bot/**"| BS["🤖 Bot Service :8090"]
    GW -->|"/api/predict/**"| PY["🐍 Python ML Service :8091"]
    GW -->|"/profile/pets/qr-scan/**"| PS
```

---

## 📁 Структура проекта

```
vet-gateway/
├── src/main/java/ru/sibsutis/vet_gateway/
│   ├── VetGatewayApplication.java     # Точка входа
│   └── configuration/
│       ├── CorsConfig.java             # CORS-политика
│       ├── LoggingGlobalFilter.java    # Глобальный фильтр логирования
│       └── SecurityConfig.java         # Валидация JWT, публичные маршруты
├── src/main/resources/
│   └── application.properties          # Маршруты и конфигурация
├── Dockerfile
└── README.md
```

---

## 🗺 Таблица маршрутов

| ID | Публичный путь | Целевой сервис | Порт | Фильтры |
|---|---|---|---|---|
| `keycloak` | `/auth/**` | Keycloak | 8080 | `RewritePath=/auth/(?<segment>.*), /${segment}` |
| `appointment-service` | `/api/appointment/**` | Appointment Service | 8081 | `StripPrefix=1` |
| `management-service` | `/api/management/**` | Management Service | 8082 | `StripPrefix=1` |
| `profile-service` | `/api/profile/**` | Profile Service | 8083 | `StripPrefix=1` |
| `pet-health-analyzer-service` | `/api/health/**` | Pet Health Analyzer | 8084 | `StripPrefix=1` |
| `telegram-bot` | `/bot/**` | Bot Service | 8090 | `StripPrefix=1` |
| `python-ai` | `/api/predict` | Python ML Service | 8091 | `StripPrefix=1` |
| `profile-service-qr` | `/profile/pets/qr-scan/**` | Profile Service | 8083 | — (публичный) |

> 💡 Обратите внимание: маршрут `profile-service-qr` **не содержит префикс `/api`** — это сделано специально, чтобы QR-ссылка, которую человек видит в браузере, была короткой и «человекопонятной»: `http://<host>/profile/pets/qr-scan/{code}`.

---

## 🔐 Безопасность

### Публичные маршруты

В `SecurityConfig` явно разрешены:

- `OPTIONS /**` — предварительные CORS-запросы (preflight);
- `GET /profile/pets/qr-scan/**` — карточка питомца по QR-коду, доступная без авторизации (сценарий «нашли потерявшееся животное»).

Все остальные запросы требуют валидного JWT.

### Валидация JWT

Конфигурация ресурсного сервера:

```properties
spring.security.oauth2.resourceserver.jwt.issuer-uri=${KEYCLOAK_ISSUER_URI}
spring.security.oauth2.resourceserver.jwt.jwk-set-uri=${KEYCLOAK_ISSUER_URI}/protocol/openid-connect/certs
```

Gateway сам скачивает публичные ключи с Keycloak, кэширует их и проверяет подпись каждого токена. Бизнес-сервисы получают уже аутентифицированные запросы и могут доверять заголовкам.

### CORS

`CorsConfig` регистрирует `CorsWebFilter` для всех путей. Допустимый origin задаётся переменной окружения:

```properties
frontend.base.url=${FRONTEND_URL:http://localhost:3000}
```

Такой подход позволяет без пересборки образа переключаться между локальным фронтендом, стейджингом и production.

---

## 📊 Логирование

Глобальный фильтр `LoggingGlobalFilter` фиксирует заголовки всех входящих запросов. Пример лога:

```
INFO  o.s.c.g.filter.LoggingGlobalFilter - Request headers: [host:"localhost:8888", 
  authorization:"Bearer eyJhbGciOiJSUzI1NiIs...", user-agent:"Mozilla/5.0 ..."]
```

Уровни логирования Spring Cloud Gateway и Spring Security настраиваются через `application.properties`:

```properties
logging.level.org.springframework.cloud.gateway=DEBUG
logging.level.org.springframework.security=DEBUG
```

В production эти уровни стоит понизить до `INFO` / `WARN`, чтобы не засорять логи.

---

## ⚙️ Конфигурация

### Переменные окружения

| Переменная | Назначение | Значение по умолчанию |
|---|---|---|
| `FRONTEND_URL` | Разрешённый origin для CORS | `http://localhost:3000` |
| `KEYCLOAK_HOST` / `KEYCLOAK_PORT` | Адрес Keycloak | `localhost` / `8080` |
| `KEYCLOAK_ISSUER_URI` | Issuer URI для JWT | `http://localhost:8080/realms/clinic-realm` |
| `APPOINTMENT_HOST` / `APPOINTMENT_PORT` | Appointment Service | `localhost` / `8081` |
| `MANAGEMENT_HOST` / `MANAGEMENT_PORT` | Management Service | `localhost` / `8082` |
| `PROFILE_HOST` / `PROFILE_PORT` | Profile Service | `localhost` / `8083` |
| `HEALTH_HOST` / `HEALTH_PORT` | Pet Health Analyzer | `localhost` / `8084` |
| `TELEGRAM_HOST` / `TELEGRAM_PORT` | Bot Service | `localhost` / `8090` |
| `PYTHON_HOST` / `PYTHON_PORT` | Python ML Service | `localhost` / `8091` |

---

## 🚀 Запуск

### Локально (Docker Compose)

```bash
docker compose up -d vet-gateway
```

### Запуск только сервиса (для разработки)

```bash
cd services/vet-gateway
./gradlew bootRun
```

### Сборка образа

```bash
docker build -t vet-gateway:latest .
```

### Проверка работоспособности

```bash
# Health check
curl http://localhost:8888/actuator/health

# Маршрутизация к публичному QR-эндпоинту
curl http://localhost:8888/profile/pets/qr-scan/{code}

# Авторизованный запрос (JWT обязателен)
curl -H "Authorization: Bearer $TOKEN" \
     http://localhost:8888/api/profile/owners
```

---

## 🔗 Связанные сервисы

| Сервис | Роль |
|---|---|
| **Keycloak** | Источник JWT и JWK-ключей для валидации |
| **Profile / Management / Appointment / Health** | Целевые сервисы маршрутизации |
| **Bot Service** | Внутренний вызов из Gateway через `/bot/**` |
| **Python ML Service** | Приём запросов на предсказание через `/api/predict` |
| **Web Frontend** | Внешний клиент, источник CORS-запросов |

---
