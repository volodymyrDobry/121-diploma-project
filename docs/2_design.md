# Розділ 2. Проєктування та обґрунтування архітектурних рішень

Проєктування архітектури системи «Шипшина» за методологією Docs-as-Code. Усі діаграми виконано як
Diagrams-as-Code (Mermaid.js) і рендеряться безпосередньо в інтерфейсі GitHub. Документ побудовано
з наскрізним трасуванням вимог ([`srs.md`](srs.md)) в архітектуру та рішення ([`adr/`](adr/)).

## 2.1 Обґрунтування архітектурного стилю та патернів

**Макро-стиль.** Для системи обрано **модульний моноліт на основі event-driven взаємодії** з
винесенням навантажених компонентів (трекінг телеметрії, планувальник маршрутів, OSINT) у окремі
сервіси. Такий вибір технічно забезпечує виконання NFR-вимог: горизонтальне масштабування трекінгу
та планувальника незалежно одне від одного задовольняє [SRS-NFR-PERF-03](srs.md#srs-nfr-perf-03)
(≥ 300 одночасних рейсів), а асинхронна шина подій (Apache Kafka) розв'язує синхронну залежність
«зміна загрози → перерахунок маршрутів» і підтримує [SRS-NFR-PERF-04](srs.md#srs-nfr-perf-04)
(перерахунок ≤ 5 с). Внутрішня структура сервісів підпорядкована принципам **Clean Architecture**
та **DDD**: доменне ядро ізольоване від інфраструктури, що знижує зв'язність (Coupling) і робить
бізнес-логіку незалежною від СУБД та брокера.

**Мікро-патерни (рівень коду).** У критичних сервісах застосовано:

- **Repository** — ізоляція доступу до даних (`RouteRepository`), доменне ядро не залежить від JDBC;
- **Dependency Injection / Dependency Inversion** — залежності спрямовані до ядра, зовнішні системи
  підключаються через інтерфейси-клієнти (`ThreatQueryClient`, `WeatherClient`);
- **Strategy** — вибір політики маршруту (`RoutePolicySelector`: безпека / час / заряд) без
  розгалужень у ядрі;
- **Observer** — публікація подій (`RouteEventPublisher`) розв'язує сервіси через шину;
- **Circuit Breaker** — запобігає каскадним збоям при тимчасовій недоступності сервісу
  ([SRS-NFR-REL-02](srs.md#srs-nfr-rel-02)): відкриття кола ізолює відмову й повертає graceful
  degradation замість блокування потоку.

## 2.2 Контекстне та контейнерне моделювання системи (C4 Model: Levels 1–2)

### C4 Level 1 — System Context Diagram

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'IBM Plex Sans, Segoe UI, sans-serif','fontSize':'15px','primaryColor':'#FBF8F2','primaryTextColor':'#221C18','primaryBorderColor':'#9E2B37','lineColor':'#8C8276'}}}%%
flowchart LR
    subgraph users[" "]
        direction TB
        OP(["👤 Оператор"])
        AN(["👤 Аналітик"])
        AD(["👤 Адміністратор"])
    end
    SYS["<b>Шипшина</b><br/>планування та моніторинг<br/>доставки вантажів БПЛА"]
    subgraph ext["Зовнішні системи"]
        direction TB
        DR["Борт БПЛА"]
        ME["Метео-API"]
        OS["OSINT-джерела"]
        TI["Картографія"]
    end
    OP -- "HTTPS/REST" --> SYS
    AN -- "HTTPS/REST" --> SYS
    AD -- "HTTPS/REST · WSS" --> SYS
    DR -- "MQTT/TCP · TLS" --> SYS
    OS -- "HTTPS/REST" --> SYS
    SYS -- "HTTPS/REST" --> ME
    SYS -- "HTTPS" --> TI
    classDef person fill:#F0D9D6,stroke:#9E2B37,stroke-width:1.5px,color:#221C18
    classDef sys fill:#9E2B37,stroke:#7a1f28,stroke-width:2px,color:#FBF8F2,font-weight:bold
    classDef extn fill:#EFE9DF,stroke:#8C8276,stroke-width:1px,color:#221C18
    class OP,AN,AD person
    class SYS sys
    class DR,ME,OS,TI extn
    style users fill:transparent,stroke:transparent
    style ext fill:#F3EFE8,stroke:#DDD4C7,stroke-dasharray:4 3
```

Система «Шипшина» розглядається як єдиний блок у взаємодії з ролями користувачів (Оператор,
Аналітик, Адміністратор) через HTTPS/REST та WSS, приймає телеметрію з борту БПЛА (MQTT/TCP, TLS)
і звертається до зовнішніх сервісів погоди, OSINT та картографії (HTTPS/REST).

### C4 Level 2 — Container Diagram

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'IBM Plex Sans, Segoe UI, sans-serif','fontSize':'14px','primaryColor':'#FBF8F2','primaryTextColor':'#221C18','primaryBorderColor':'#9E2B37','lineColor':'#8C8276'}}}%%
flowchart TB
    OP(["👤 Оператор"]):::person
    DRONE["Борт БПЛА"]:::extn
    METEO["Метео-API"]:::extn

    subgraph edge["① Клієнт та Edge"]
        direction LR
        SPA["Веб-застосунок<br/><i>React / TypeScript</i>"]:::app
        GW["API Gateway<br/><i>Spring Cloud Gateway</i>"]:::app
        IDP["Identity Provider<br/><i>Keycloak · OIDC</i>"]:::app
    end

    subgraph services["② Сервіси застосунку · Java / Spring Boot"]
        direction LR
        DEL["Delivery"]:::svc
        RP["Route Planning"]:::svc
        TR["Tracking"]:::svc
        WD["Silence Watchdog"]:::svc
        TH["Threat"]:::svc
        OSS["OSINT · Python"]:::svc
    end

    BUS{{"Apache Kafka · шина подій"}}:::bus

    subgraph data["③ Сховища даних"]
        direction LR
        PG[("PostgreSQL<br/>+ PostGIS")]:::db
        TS[("TimescaleDB")]:::db
        RD[("Redis")]:::db
    end

    OP -- "HTTPS" --> SPA
    SPA -- "REST/HTTPS · WSS" --> GW
    GW -. "OIDC/HTTPS" .-> IDP
    GW -- "REST/HTTP" --> DEL
    GW -- "REST/HTTP" --> RP
    GW -- "REST/HTTP" --> TH
    GW -- "WSS" --> TR
    DRONE -- "MQTT/TCP" --> TR
    DEL & RP & TR & WD & TH & OSS <== "події" ==> BUS
    DEL -- "JDBC" --> PG
    RP -- "JDBC" --> PG
    TH -- "JDBC · ST_DWithin" --> PG
    TR -- "JDBC" --> TS
    TR & RP -- "RESP" --> RD
    RP -. "HTTPS/REST" .-> METEO

    classDef person fill:#F0D9D6,stroke:#9E2B37,stroke-width:1.5px,color:#221C18
    classDef app fill:#E7EEF0,stroke:#2F5D63,stroke-width:1.5px,color:#13302f
    classDef svc fill:#FBF8F2,stroke:#9E2B37,stroke-width:1.5px,color:#221C18
    classDef db fill:#EAF3EC,stroke:#2E7D5B,stroke-width:1.5px,color:#13342a
    classDef bus fill:#F6E8CC,stroke:#C08A2E,stroke-width:2px,color:#4a3410,font-weight:bold
    classDef extn fill:#EFE9DF,stroke:#8C8276,stroke-width:1px,color:#221C18
    style edge fill:#F3EFE8,stroke:#DDD4C7
    style services fill:#FAF6EF,stroke:#DDD4C7
    style data fill:#F3EFE8,stroke:#DDD4C7
```

**Контейнери та протоколи.** Веб-застосунок (React/TS) взаємодіє з API Gateway через REST/HTTPS і
WSS. API Gateway (Spring Cloud Gateway) виконує маршрутизацію (REST/HTTP), валідацію токенів через
Identity Provider (OIDC/HTTPS) та rate limiting. Бізнес-сервіси (Java/Spring Boot; OSINT — Python)
взаємодіють асинхронно через Apache Kafka (Kafka/TCP). Сховища: PostgreSQL+PostGIS (JDBC/TCP, SQL;
у Threat Service — ST_DWithin), TimescaleDB (треки), Redis (RESP/TCP).

**Трасування SRS → контейнери:** [SRS-FUN-01](srs.md#srs-fun-01) → Delivery; SRS-FUN-02/03 → Route
Planning; [SRS-FUN-04](srs.md#srs-fun-04) → Tracking; SRS-FUN-05 → Silence Watchdog; SRS-FUN-06/07 →
Threat (+ OSINT); [SRS-FUN-08](srs.md#srs-fun-08) → API Gateway + Identity Provider.

## 2.3 Компонентне моделювання та опис серверного ядра (C4 Model: Level 3)

Критичним сервісом для декомпозиції обрано **Route Planning Service**.

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'IBM Plex Sans, Segoe UI, sans-serif','fontSize':'14px','primaryColor':'#FBF8F2','primaryTextColor':'#221C18','primaryBorderColor':'#9E2B37','lineColor':'#8C8276'}}}%%
flowchart TB
    subgraph edge["Зовнішні · вхід"]
        direction LR
        GW["API Gateway"]:::extn
        KIN["Apache Kafka<br/><i>threat.added · weather.updated</i>"]:::extn
    end

    subgraph inb["① Inbound"]
        direction LR
        GUARD["AuthGuard<br/><i>JWT · RBAC</i>"]:::xcut
        VAL["RequestValidator<br/><i>DTO</i>"]:::xcut
        CTRL["RoutePlanningController<br/><i>REST</i>"]:::in
        CONS["RouteRecalcConsumer<br/><i>Kafka listener</i>"]:::in
    end

    subgraph core["② Core · бізнес-логіка"]
        direction LR
        SVC["RoutePlanningService<br/><i>оркестратор</i>"]:::coreMain
        POL["RoutePolicySelector"]:::core
        GRAPH["RiskGraphBuilder"]:::core
        ASTAR["AStarPathfinder"]:::core
        PD["PDoletuEstimator<br/><i>−ln(p)</i>"]:::core
    end

    subgraph outb["③ Outbound"]
        direction LR
        TQC["ThreatQueryClient"]:::out
        WC["WeatherClient"]:::out
        REPO["RouteRepository"]:::out
        CACHE["RouteCache"]:::out
        PUB["RouteEventPublisher<br/><i>Observer</i>"]:::out
    end

    subgraph data["Сховища та шина · вихід"]
        direction LR
        PG[("PostgreSQL+PostGIS")]:::db
        RD[("Redis")]:::db
        KOUT["Apache Kafka<br/><i>route.completed</i>"]:::extn
    end

    GW --> GUARD --> VAL --> CTRL --> SVC
    KIN --> CONS --> SVC
    SVC --> POL & GRAPH & ASTAR
    ASTAR --> PD
    GRAPH --> TQC & WC
    SVC --> REPO & CACHE & PUB
    REPO --> PG
    CACHE --> RD
    PUB -. "route.completed" .-> KOUT

    classDef in fill:#E7EEF0,stroke:#2F5D63,stroke-width:1.5px,color:#13302f
    classDef xcut fill:#F0D9D6,stroke:#9E2B37,stroke-width:1.2px,color:#221C18
    classDef coreMain fill:#9E2B37,stroke:#7a1f28,stroke-width:2px,color:#FBF8F2,font-weight:bold
    classDef core fill:#FBF8F2,stroke:#9E2B37,stroke-width:1.3px,color:#221C18
    classDef out fill:#F6E8CC,stroke:#C08A2E,stroke-width:1.3px,color:#4a3410
    classDef db fill:#EAF3EC,stroke:#2E7D5B,stroke-width:1.5px,color:#13342a
    classDef extn fill:#EFE9DF,stroke:#8C8276,stroke-width:1px,color:#221C18
    style edge fill:#F3EFE8,stroke:#DDD4C7
    style inb fill:#F3EFE8,stroke:#DDD4C7
    style core fill:#FAF6EF,stroke:#DDD4C7
    style outb fill:#F3EFE8,stroke:#DDD4C7
    style data fill:#F3EFE8,stroke:#DDD4C7
```

**Шари Clean Architecture.** Inbound: `RoutePlanningController` (REST) та `RouteRecalcConsumer`
(Kafka); вхідний трафік проходить крізь `AuthGuard` (JWT/RBAC) і `RequestValidator` (DTO). Core:
`RoutePlanningService` (оркестратор), `RiskGraphBuilder`, `AStarPathfinder`, `PDoletuEstimator`,
`RoutePolicySelector`. Outbound: `ThreatQueryClient`, `WeatherClient`, `RouteRepository`,
`RouteCache`, `RouteEventPublisher` (Observer). Cross-cutting: `MetricsInterceptor`,
`GlobalErrorHandler`. Контролери не звертаються напряму до репозиторіїв — доступ до даних лише через
Service, що забезпечує ізоляцію доменного ядра.

## 2.4 Контур безпеки програмного рішення (Application Security)

Контур AppSec реалізує оборону за принципом defense-in-depth на рівні коду серверного ядра та
задовольняє [SRS-NFR-SEC-01..03](srs.md#srs-nfr-sec-01).

- **Автентифікація та сесії.** OAuth 2.0 / OIDC (Keycloak); на вході сервісу `AuthGuard` валідує
  підпис JWT (RS256) за публічним ключем JWKS; access-токен TTL ≤ 15 хв, refresh — rotation.
- **Хешування секретів.** Паролі службових акаунтів — **Argon2id** (memory-hard, параметри
  m=64 MiB, t=3, p=1); жодних самописних схем шифрування.
- **Валідація вхідних даних.** `ValidationPipe` на рівні DTO блокує Mass Assignment (whitelist
  полів), перевіряє `minimum`/`maximum`/`pattern`; відхиляє не описані в контракті поля.
- **Авторизація.** RBAC/ABAC `Guards` на кожному контролері; перевірка прав за роллю
  (operator/analyst/admin) та власністю ресурсу (ABAC) запобігає BOLA/IDOR.
- **Захист периметра.** Rate Limiting на API Gateway (напр. 100 req/хв на токен) проти Brute-Force /
  DoS; CORS-політика з явним allow-list походжень; обмеження Max Body Size на завантаження.
- **Аудит та спостережуваність безпеки.** `Audit Logging` усіх дій із загрозами й рейсами у
  незмінний журнал; інциденти безпеки (невдалі автентифікації, 401/403) експортуються в OTel.
- **Покриття OWASP API Security Top-10:** BOLA → ABAC Guards; Broken Auth → OIDC+JWT; Mass
  Assignment → ValidationPipe; відсутність rate limiting → Gateway; security misconfiguration →
  SAST (SonarQube) + сканування образів (Trivy) у CI ([SRS-NFR-SEC-03](srs.md#srs-nfr-sec-03)).

## 2.5 Специфікація міжсервісних контрактів та топологія API-маршрутів

Повну специфікацію інтерфейсів оформлено за підходом API-First Design у файлі
[`api/openapi.yaml`](api/openapi.yaml) (OpenAPI 3.0). Контракт містить схеми запитів/відповідей,
схему авторизації `bearerAuth` (JWT) та структуру обробки помилок (4xx/5xx).

### Карта інтерфейсних маршрутів системи (API topology)

| ID | Метод / RPC | Топологічний шлях (URI) | Вхідні параметри / Payload | Контейнер-обробник |
|---|---|---|---|---|
| API-REQ-01 | POST | `/api/v1/deliveries` | `CreateDeliveryRequest` (body) | Delivery Service |
| API-REQ-02 | POST | `/api/v1/deliveries/{id}/routes` | `PlanRoutesRequest` (body, path id) | Route Planning Service |
| API-REQ-03 | GET | `/api/v1/flights/{id}/track` | path id; query `from`, `to` | Tracking Service |
| API-REQ-04 | POST | `/api/v1/threats` | `ThreatEmplacementRequest` (body) | Threat Service |
| API-REQ-05 | GET | `/api/v1/threats?bbox=` | query `bbox`, `page`, `limit` (≤ 100) | Threat Service |
| API-EVT-06 | Kafka | topic `telemetry.received` | `TelemetryPoint` (Avro) | Tracking → Kafka |

### Потоки даних (Data Flows)

**DF-1. Створення доставки → маршрут.** Крок 1: клієнтський Frontend надсилає `CreateDeliveryRequest`
через HTTPS POST на `/api/v1/deliveries`. Крок 2: API Gateway перехоплює запит, валідує JWT
(bearerAuth) та застосовує rate limit. Крок 3: Delivery Service зберігає доставку (JDBC, транзакція)
та публікує `delivery.created` у Kafka. Крок 4: Route Planning Service будує маршрути й повертає
масив із `p_delivery` для кожного.

**DF-2. Телеметрія в реальному часі.** Борт БПЛА → Tracking Service (MQTT/TCP) → topic
`telemetry.received` (Kafka) → споживачі Tracking (WSS до оператора), Silence Watchdog (таймери),
Analytics. Обробка ідемпотентна за `(flightId, seq)` ([SRS-NFR-REL-03](srs.md#srs-nfr-rel-03)).

**Мережеві обмеження:** Max Body Size запиту — 2 MiB (10 MiB для завантаження файлів); пагінація
обов'язкова для списків (`limit` ≤ 100); CORS allow-list; усі маршрути, що змінюють стан
(POST/PUT/DELETE), вимагають Bearer-токен.

## 2.6 Моделювання користувацького досвіду (Use Cases)

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'IBM Plex Sans, Segoe UI, sans-serif','fontSize':'14px','primaryColor':'#FBF8F2','primaryTextColor':'#221C18','primaryBorderColor':'#9E2B37','lineColor':'#8C8276'}}}%%
flowchart LR
    OP(["👤 Оператор"]):::actor
    AN(["👤 Аналітик"]):::actor
    AD(["👤 Адміністратор"]):::actor
    IDP["⚙ Identity Provider"]:::sactor
    DRONE["⚙ Борт БПЛА"]:::sactor

    subgraph SYS["Межі системи «Шипшина»"]
        direction TB
        UC1(["UC-01<br/>Створити доставку"]):::uc
        UC2(["UC-02<br/>Побудувати маршрути"]):::uc
        UC3(["UC-03<br/>Запустити рейс"]):::uc
        UC4(["UC-04<br/>Моніторити рейс"]):::uc
        UC5(["UC-05<br/>Опрацювати втрату зв'язку"]):::uc
        UC6(["UC-06<br/>Вести реєстр загроз"]):::uc
        AUTH(["Автентифікуватися"]):::ucinc
        ALERT(["Сповістити про інцидент"]):::ucext
    end

    OP --- UC1
    OP --- UC2
    OP --- UC3
    OP --- UC4
    AN --- UC6
    AN --- UC5
    AD --- UC6
    UC1 -. "&laquo;include&raquo;" .-> AUTH
    UC6 -. "&laquo;include&raquo;" .-> AUTH
    UC4 -. "&laquo;extend&raquo;" .-> ALERT
    UC5 -. "&laquo;extend&raquo;" .-> ALERT
    AUTH --- IDP
    UC4 --- DRONE
    UC2 --- UC6

    classDef actor fill:#F0D9D6,stroke:#9E2B37,stroke-width:1.5px,color:#221C18,font-weight:bold
    classDef sactor fill:#EFE9DF,stroke:#8C8276,stroke-width:1px,color:#221C18
    classDef uc fill:#FBF8F2,stroke:#2F5D63,stroke-width:1.5px,color:#13302f
    classDef ucinc fill:#E7EEF0,stroke:#2F5D63,stroke-width:1.3px,color:#13302f
    classDef ucext fill:#F6E8CC,stroke:#C08A2E,stroke-width:1.3px,color:#4a3410
    style SYS fill:#F3EFE8,stroke:#9E2B37,stroke-width:1.5px,stroke-dasharray:6 4
```

### Специфікації прецедентів

**UC-01 — Створити доставку.** Primary Actor: Оператор. Secondary: Identity Provider.
*Preconditions:* користувач автентифікований (роль operator); є ≥ 1 доступний тип дрона.

| Крок | Дія Актора | Реакція та відповідь системи |
|---|---|---|
| 1 | Задає точки призначення та параметри посилки | `ValidationPipe` перевіряє DTO; підбирає сумісний `container_class` (SQL-запит) |
| 2 | Підтверджує створення | Delivery Service відкриває транзакцію, зберігає `status := DRAFT`, публікує `delivery.created` |

*Postconditions:* доставка у стані DRAFT; подію опубліковано в Kafka.

**AF-01A (Alternative, UC-01).** Старт із кроку 1. Умова-тригер: обраний тип дрона не покриває
дальність до точки призначення. Обробка: система відхиляє підбір, пропонує сумісні типи дрона;
повернення на крок 1.

**EF-01A (Exception, UC-02 — побудова маршруту).** Умова: Threat Service повернув timeout під час
просторового запиту. Технічні дії сервера: circuit breaker розмикає коло після N збоїв, запит
повторюється (retry з backoff) до 3 разів; за вичерпання — повернення помилки `503` з кодом
`ROUTE_DEPENDENCY_UNAVAILABLE`, транзакція не фіксується (Rollback), стан системи узгоджений.

## 2.7 Проєктування та документування моделей даних (ERD)

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'IBM Plex Sans, Segoe UI, sans-serif','fontSize':'13px','primaryColor':'#FBF8F2','primaryTextColor':'#221C18','primaryBorderColor':'#9E2B37','lineColor':'#8C8276'}}}%%
erDiagram
    USERS ||--o{ DELIVERIES : "створює"
    DRONE_TYPES ||--o{ DELIVERIES : "призначається"
    CONTAINER_CLASSES ||--o{ DELIVERIES : "підбирається"
    DELIVERIES ||--o{ ROUTES : "має"
    DELIVERIES ||--o{ FLIGHTS : "породжує"
    ROUTES ||--o| FLIGHTS : "виконується"
    FLIGHTS ||--o{ TELEMETRY_POINTS : "надсилає"
    FLIGHTS ||--o{ INCIDENTS : "фіксує"
    THREAT_TYPES ||--o{ THREAT_EMPLACEMENTS : "розміщується"

    USERS {
        uuid id PK
        varchar email UK
        varchar role "ENUM operator|analyst|admin"
        timestamptz created_at
    }
    DRONE_TYPES {
        uuid id PK
        varchar name
        int range_km
        int cruise_speed_kmh
        numeric payload_kg
        int ceiling_m
        smallint jamming_resistance
    }
    CONTAINER_CLASSES {
        uuid id PK
        varchar name
        numeric max_weight_kg
        jsonb dimensions
    }
    DELIVERIES {
        uuid id PK
        uuid created_by FK
        uuid drone_type_id FK
        uuid container_class_id FK
        varchar status "ENUM DRAFT|PLANNED|IN_FLIGHT|DELIVERED|FAILED|LOST"
        geography origin "Point 4326"
        geography destination "Point 4326"
        timestamptz window_start
        timestamptz window_end
        timestamptz created_at
        timestamptz updated_at
        int version "optimistic lock"
    }
    ROUTES {
        uuid id PK
        uuid delivery_id FK
        numeric p_delivery "0..1"
        int eta_sec
        numeric energy_pct
        geography geom "LineString 4326"
        varchar status
        timestamptz created_at
    }
    FLIGHTS {
        uuid id PK
        uuid delivery_id FK
        uuid route_id FK
        varchar state "ENUM ACTIVE|SILENT|PRESUMED_LOST|SHOT_DOWN|DELIVERED"
        geography last_known_point "Point 4326"
        timestamptz started_at
        timestamptz ended_at
    }
    TELEMETRY_POINTS {
        bigint id PK
        uuid flight_id FK
        bigint seq "idempotency"
        timestamptz ts "hypertable key"
        geography position "Point 4326"
        int altitude_m
        numeric speed_kmh
        numeric battery_pct
        smallint link_quality
    }
    THREAT_TYPES {
        uuid id PK
        varchar name
        varchar category "ENUM REB|PPO"
        int effective_radius_m
    }
    THREAT_EMPLACEMENTS {
        uuid id PK
        uuid threat_type_id FK
        geography location "Point 4326"
        numeric confidence "0..1"
        varchar source "ENUM manual|osint"
        boolean active
        timestamptz last_confirmed_at
    }
    INCIDENTS {
        uuid id PK
        uuid flight_id FK
        varchar kind "ENUM SILENT|PRESUMED_LOST|SHOT_DOWN"
        geography last_known_point "Point 4326"
        jsonb cause_candidates
        timestamptz created_at
    }
```

**Специфікація сутностей (витяг).**

- **DELIVERIES** — ядро транзакційної моделі. PK `id` (uuid). FK: `created_by`→USERS,
  `drone_type_id`→DRONE_TYPES, `container_class_id`→CONTAINER_CLASSES. `status` — тип ENUM (а не
  VARCHAR) для цілісності станів. `origin`/`destination` — `geography(Point, 4326)`. Поле `version`
  реалізує optimistic locking; `created_at`/`updated_at` — аудит.
- **THREAT_EMPLACEMENTS** — розміщення загроз. `location` — `geography(Point, 4326)` з **GiST-індексом**
  для запиту «загрози в коридорі» ([SRS-FUN-07](srs.md#srs-fun-07)). `source` ENUM `manual|osint`
  розрізняє походження; `confidence` та `last_confirmed_at` — свіжість даних.
- **TELEMETRY_POINTS** — гіпертаблиця TimescaleDB, ключ партиціювання `ts`. `seq` забезпечує
  ідемпотентність; індекс за `(flight_id, ts)`.

**План індексації:** GiST на всіх `geography`-полях (B-Tree недоцільний для просторових запитів);
B-Tree на FK та `status`; BRIN на `telemetry_points.ts` (великий монотонний часовий ряд).

## 2.8 Моделювання логіки критичних вузлів

### Алгоритм 1 — Risk-aware A* (побудова маршруту з P_долету)

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'IBM Plex Mono, monospace','fontSize':'13px','primaryColor':'#FBF8F2','primaryTextColor':'#221C18','primaryBorderColor':'#2F5D63','lineColor':'#8C8276'}}}%%
flowchart TD
    S(["Початок: Risk-aware A*"]):::term
    I["Вхід: сітка G, старт s, ціль t,<br/>ваги λ_risk, λ_wind"]:::io
    INIT["open := перша-черга · s<br/>g[s] := 0<br/>f[s] := h(s, t)"]:::proc
    Q{"open НЕ порожня?"}:::dec
    POP["u := argmin f серед open<br/>видалити u з open"]:::proc
    GOAL{"u = t ?"}:::dec
    FOR{"Є необроблене<br/>ребро e = (u, v) ?"}:::dec
    COST["p_e := ∏ виживання по зонах(v)<br/>w := довжина[e] + λ_risk · (−ln p_e)<br/>&nbsp;&nbsp;&nbsp;&nbsp;+ λ_wind · вітер[e]"]:::proc
    REL{"g[u] + w  МЕНШЕ  g[v] ?"}:::dec
    UPD["g[v] := g[u] + w<br/>f[v] := g[v] + h(v, t)<br/>open := open ∪ v"]:::proc
    OK(["Вихід: маршрут + P_долету = exp(−g[t])"]):::termok
    NO(["Вихід: NO_ROUTE"]):::termno

    S --> I --> INIT --> Q
    Q -- "НІ" --> NO
    Q -- "ТАК" --> POP --> GOAL
    GOAL -- "ТАК" --> OK
    GOAL -- "НІ" --> FOR
    FOR -- "НІ" --> Q
    FOR -- "ТАК" --> COST --> REL
    REL -- "ТАК" --> UPD --> FOR
    REL -- "НІ" --> FOR

    classDef term fill:#9E2B37,stroke:#7a1f28,color:#FBF8F2,font-weight:bold
    classDef termok fill:#2E7D5B,stroke:#1f5a40,color:#FBF8F2,font-weight:bold
    classDef termno fill:#8C8276,stroke:#5f574d,color:#FBF8F2,font-weight:bold
    classDef io fill:#E7EEF0,stroke:#2F5D63,color:#13302f
    classDef proc fill:#FBF8F2,stroke:#9E2B37,color:#221C18
    classDef dec fill:#F6E8CC,stroke:#C08A2E,color:#4a3410
```

Логарифмічне перетворення `−ln(p_e)` робить мультиплікативну ймовірність виживання адитивною, тому
класичний A* безпосередньо максимізує `P_долету`. Евристика `h` — гаверсинус до цілі (допустима,
бо ризик лише додає вартості понад геометрію).

### Алгоритм 2 — Silence Watchdog (детекція мовчання та класифікація втрати)

```mermaid
%%{init: {'theme':'base','themeVariables':{'fontFamily':'IBM Plex Mono, monospace','fontSize':'13px','primaryColor':'#FBF8F2','primaryTextColor':'#221C18','primaryBorderColor':'#2F5D63','lineColor':'#8C8276'}}}%%
flowchart TD
    S(["Початок: Silence Watchdog · тік таймера"]):::term
    I["Вхід: рейс f, пороги T1, T2,<br/>last_seen[f], останній вектор стану"]:::io
    D["Δ := now − last_seen[f]"]:::proc
    C2{"Δ  БІЛЬШЕ  T2 ?"}:::dec
    ZONE{"остання точка В зоні загрози<br/>AND різкий обрив сигналу ?"}:::dec
    SHOT["state[f] := SHOT_DOWN"]:::procR
    LOST["state[f] := PRESUMED_LOST"]:::procR
    INC["створити інцидент(f,<br/>last_known_point, cause_candidates)"]:::proc
    C1{"Δ  БІЛЬШЕ  T1 ?"}:::dec
    SIL["state[f] := SILENT<br/>est_pos := deadReckoning(вектор)"]:::procY
    ACT["state[f] := ACTIVE"]:::procG
    E(["Кінець тіку"]):::termok

    S --> I --> D --> C2
    C2 -- "ТАК" --> ZONE
    ZONE -- "ТАК" --> SHOT --> INC
    ZONE -- "НІ" --> LOST --> INC
    INC --> E
    C2 -- "НІ" --> C1
    C1 -- "ТАК" --> SIL --> E
    C1 -- "НІ" --> ACT --> E

    classDef term fill:#9E2B37,stroke:#7a1f28,color:#FBF8F2,font-weight:bold
    classDef termok fill:#2F5D63,stroke:#1f4247,color:#FBF8F2,font-weight:bold
    classDef io fill:#E7EEF0,stroke:#2F5D63,color:#13302f
    classDef proc fill:#FBF8F2,stroke:#9E2B37,color:#221C18
    classDef procR fill:#F0D9D6,stroke:#9E2B37,color:#7a1f28,font-weight:bold
    classDef procY fill:#F6E8CC,stroke:#C08A2E,color:#4a3410
    classDef procG fill:#EAF3EC,stroke:#2E7D5B,color:#13342a
    classDef dec fill:#F6E8CC,stroke:#C08A2E,color:#4a3410
```

Двоетапні таймери `T1`/`T2` відокремлюють тимчасове мовчання каналу від остаточної втрати. Логічні
умови блоків розгалуження використовують оператори `AND`/`OR`; присвоєння позначено `:=`; індекси
масивів — `state[f]`, `last_seen[f]`.

## 2.9 Інтеграційний аудит та верифікація цілісності доменів (Production Readiness)

Проведено перехресний контроль узгодженості трьох артефактів: моделі даних (2.7), API-контрактів
(2.5) та логіки алгоритмів (2.8).

**Перевірка «API ↔ Database».** Типи полів у `openapi.yaml` звірено зі схемою БД: `p_delivery`
(`number`, 0..1) ↔ `numeric`; `status` (`enum`) ↔ ENUM у БД (а не VARCHAR); ідентифікатори —
`uuid`/`string(uuid)`; часові мітки — `string(date-time)` ↔ `timestamptz`. Розбіжностей немає.

**Перевірка «Logic ↔ API/Database».** Алгоритм маршруту оперує `p_e` та зонами загроз — відповідні
поля (`threat_emplacements.location`, `effective_radius_m`) наявні в БД та доступні через
`ThreatQueryClient`. Стан `state[f]` з алгоритму 2 відповідає ENUM `flights.state`.

**Production Readiness Report.** Специфікації Розділу 2 узгоджені й готові до реалізації (Розділ 3):
критичних блокерів інтеграції (Showstoppers) не виявлено; типи даних консистентні між шарами; усі
функціональні вимоги SRS мають архітектурний елемент-обробник; контури безпеки та спостережуваності
визначені. Статус готовності архітектурного ядра — **Approved for implementation**.

## Висновки до розділу 2

У розділі виконано наскрізне проєктування архітектури системи за методологією Docs-as-Code.
Обґрунтовано вибір модульного event-driven стилю з елементами Clean Architecture та визначено
мікро-патерни (Repository, DI, Strategy, Observer, Circuit Breaker), що безпосередньо задовольняють
нефункціональні вимоги щодо масштабованості, latency та відмовостійкості.

Побудовано ієрархічну модель C4 (рівні 1–3), специфіковано контур безпеки застосунку (OWASP,
Argon2id, Guards, Audit Logging), міжсервісні контракти (OpenAPI) з топологією маршрутів і потоками
даних, поведінковий контур (Use Cases з Happy/Alternative/Exception Flows), логіко-фізичну модель
даних (ERD з типами, ключами та планом індексації) та логіку критичних алгоритмів у формі блок-схем.

Проведений інтеграційний аудит підтвердив узгодженість типів даних і сутностей між схемою БД,
API-контрактами та логікою алгоритмів; критичних конфліктів інтеграції не виявлено. Спроєктоване
архітектурне ядро є повним, внутрішньо несуперечливим і готовим до програмної реалізації, що
становитиме предмет розділу 3.
