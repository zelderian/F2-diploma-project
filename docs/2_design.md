# 2 ПРОЄКТУВАННЯ АРХІТЕКТУРИ СИСТЕМИ

Архітектуру системи змодельовано за методологією C4 Model (Simon Brown) на трьох рівнях: System Context, Container та Component. Діаграми виконано за принципом Diagrams-as-Code у Mermaid.js із нотацією C4 (Person, Software System, Container, Component, System_Ext). Вимоги, на які посилаються описи, наведено у специфікації [srs.md](srs.md), архітектурні рішення – у каталозі [adr/](adr/).

## 2.1 Контекстне та контейнерне моделювання системи (C4 Model)

<a id="diagram-level1"></a>
### 2.1.1 Рівень 1 – System Context

Рисунок 2.1 – Діаграма контексту системи (C4 Level 1)

```mermaid
flowchart TB
    player["<b>Гравець</b><br/>[Person]<br/>Користувач ігрового клієнта UE5 (Windows)"]:::person
    operator["<b>Оператор / DevOps</b><br/>[Person]<br/>Розгортання, моніторинг, реагування на сповіщення"]:::person
    game["<b>Co-op Survival Game Platform</b><br/>[Software System]<br/>Ігровий клієнт, Authoritative Dedicated Server,<br/>Progression API, Observability"]:::system
    eos["<b>Epic Online Services</b><br/>[System_Ext]<br/>Connect (ID Token), Lobbies, JWKS"]:::ext
    aws["<b>AWS EC2 / S3 / SSM</b><br/>[System_Ext]<br/>Хостинг, резервні копії, секрети"]:::ext
    gh["<b>GitHub Actions + GHCR</b><br/>[System_Ext]<br/>CI/CD, SonarQube, Trivy, реєстр образів"]:::ext
    player -->|"Керує матчем<br/>[UE Net Protocol / UDP]"| game
    operator -->|"Дашборди, сповіщення<br/>[HTTPS]"| game
    operator -->|"Pull Request, реліз<br/>[HTTPS]"| gh
    game -->|"Автентифікація, лобі<br/>[EOS SDK / HTTPS]"| eos
    game -->|"Виконується на EC2; бекапи в S3; секрети SSM<br/>[AWS SDK / HTTPS]"| aws
    gh -->|"Розгортання Docker-образів<br/>[SSH + docker compose pull]"| aws
    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef container fill:#438dd5,stroke:#2e6295,color:#fff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#000
    classDef db fill:#438dd5,stroke:#2e6295,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff
```

Опис елементів діаграми:

| Елемент | Тип | Роль | Протокол взаємодії |
|---|---|---|---|
| Гравець | Person | Керує ігровим клієнтом, бере участь у матчі | UE Net Protocol / UDP |
| Оператор / DevOps | Person | Ініціює релізи через Pull Request, переглядає дашборди та сповіщення | HTTPS |
| Co-op Survival Game Platform | Software System | Система, що проєктується | – |
| Epic Online Services | System_Ext | Автентифікація (ID Token), лобі, JWKS ([SRS-FUN-06](srs.md#srs-fun-06), [SRS-FUN-08](srs.md#srs-fun-08)) | EOS SDK / HTTPS |
| AWS EC2 / S3 / SSM | System_Ext | Хостинг контейнерів, резервні копії ([SRS-NFR-REL-03](srs.md#srs-nfr-rel-03)), секрети ([SRS-NFR-SEC-02](srs.md#srs-nfr-sec-02)) | AWS SDK / HTTPS |
| GitHub Actions + GHCR | System_Ext | Конвеєр CI/CD ([SRS-FUN-15](srs.md#srs-fun-15)), реєстр образів | HTTPS; SSH для розгортання |

<a id="diagram-level2"></a>
### 2.1.2 Рівень 2 – Container

Рисунок 2.2 – Контейнерна діаграма системи (C4 Level 2)

```mermaid
flowchart TB
    player["<b>Гравець</b><br/>[Person]"]:::person
    operator["<b>Оператор / DevOps</b><br/>[Person]"]:::person
    eos["<b>Epic Online Services</b><br/>[System_Ext]<br/>Connect, Lobbies, JWKS"]:::ext
    subgraph sys["Co-op Survival Game Platform [Software System]"]
        client["<b>Game Client</b><br/>[Container: UE5, C++/Blueprints]<br/>Рендеринг, ввід, Client-Side Prediction, UI лобі"]:::container
        dserver["<b>Dedicated Server</b><br/>[Container: UE5 C++, Linux, Docker]<br/>Авторитетна симуляція, валідація RPC,<br/>Spawn Director, /metrics, /healthz"]:::container
        proxy["<b>Reverse Proxy</b><br/>[Container: Caddy, Docker]<br/>TLS termination, IP rate limiting"]:::container
        api["<b>Progression API</b><br/>[Container: Go 1.23, chi, Docker]<br/>Профілі, MatchResult, ідемпотентність, JWT"]:::container
        pg[("<b>PostgreSQL 16</b><br/>[Container: Database]<br/>Профілі, статистика, інвентар, MatchResult")]:::db
        redis[("<b>Redis 7</b><br/>[Container: Cache]<br/>Кеш профілів, ключі ідемпотентності,<br/>лічильники rate limit")]:::db
        prom["<b>Prometheus</b><br/>[Container: Docker]<br/>Збір метрик кожні 15 с, правила сповіщень"]:::container
        graf["<b>Grafana</b><br/>[Container: Docker]<br/>Дашборди, Alerting"]:::container
        backup["<b>Backup Job</b><br/>[Container: cron + pg_dump]<br/>Щоденне резервне копіювання"]:::container
    end
    s3["<b>Amazon S3</b><br/>[System_Ext]"]:::ext
    gh["<b>GitHub Actions + GHCR</b><br/>[System_Ext]<br/>Build, Tests, SonarQube, Trivy"]:::ext
    player -->|"Ввід / відображення<br/>[Desktop UI]"| client
    client -->|"Login, Lobbies<br/>[EOS SDK / HTTPS]"| eos
    client -->|"Replication, Server RPC<br/>[UE Net Protocol / UDP 7777]"| dserver
    client -->|"GET /v1/players/me<br/>[REST / HTTPS + JWT]"| proxy
    dserver -->|"POST /v1/matches<br/>[REST / HTTPS + Idempotency-Key]"| proxy
    dserver -->|"Отримання JWKS<br/>[HTTPS]"| eos
    proxy -->|"Проксування<br/>[HTTP/1.1 / TCP 8080]"| api
    api -->|"Транзакції<br/>[SQL / TCP 5432, pgx]"| pg
    api -->|"Кеш, ідемпотентність<br/>[RESP / TCP 6379]"| redis
    api -->|"Отримання JWKS<br/>[HTTPS]"| eos
    prom -->|"Scrape /metrics<br/>[HTTP / TCP 9100]"| dserver
    prom -->|"Scrape /metrics<br/>[HTTP / TCP 8080]"| api
    graf -->|"PromQL<br/>[HTTP / TCP 9090]"| prom
    operator -->|"Дашборди<br/>[HTTPS]"| graf
    backup -->|"pg_dump<br/>[SQL / TCP 5432]"| pg
    backup -->|"Вивантаження дампу<br/>[HTTPS / AWS SDK]"| s3
    gh -->|"Deploy образів<br/>[SSH + docker compose pull]"| dserver
    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef container fill:#438dd5,stroke:#2e6295,color:#fff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#000
    classDef db fill:#438dd5,stroke:#2e6295,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff
```

Опис контейнерів та трасування вимог:

- **Game Client** (UE5, C++/Blueprints) – рендеринг, обробка вводу, Client-Side Prediction, UI лобі. Реалізує [SRS-FUN-08](srs.md#srs-fun-08) та [SRS-FUN-09](srs.md#srs-fun-09). Взаємодіє з Dedicated Server через UE Net Protocol / UDP 7777, з EOS – через EOS SDK / HTTPS, з Progression API – через REST / HTTPS з JWT.
- **Dedicated Server** (UE5 C++, Linux, Docker) – авторитетна симуляція матчу. Реалізує [SRS-FUN-01](srs.md#srs-fun-01)–[SRS-FUN-07](srs.md#srs-fun-07), [SRS-FUN-16](srs.md#srs-fun-16), [SRS-FUN-17](srs.md#srs-fun-17); модель розміщення обґрунтовано в [ADR-001](adr/0001-choice-of-technology.md). Внутрішню структуру наведено в [C4 Component](#diagram-level3).
- **Reverse Proxy** (Caddy) – завершення TLS 1.2+ та обмеження частоти на рівні IP перед Progression API ([SRS-NFR-SEC-02](srs.md#srs-nfr-sec-02), [SRS-NFR-SEC-03](srs.md#srs-nfr-sec-03)).
- **Progression API** (Go 1.23, chi) – REST API профілів і результатів матчу з перевіркою JWT. Реалізує [SRS-FUN-10](srs.md#srs-fun-10)–[SRS-FUN-14](srs.md#srs-fun-14), вимоги [SRS-NFR-PERF-03](srs.md#srs-nfr-perf-03), [SRS-NFR-SEC-01](srs.md#srs-nfr-sec-01), [SRS-NFR-COMP-01](srs.md#srs-nfr-comp-01).
- **PostgreSQL 16** – транзакційне сховище профілів, інвентарю та MatchResult ([SRS-FUN-10](srs.md#srs-fun-10)); доступ через SQL / TCP 5432 (драйвер pgx).
- **Redis 7** – кеш профілів ([SRS-NFR-SCAL-02](srs.md#srs-nfr-scal-02)), ключі ідемпотентності з TTL 24 год ([SRS-FUN-11](srs.md#srs-fun-11)), лічильники rate limit ([SRS-NFR-SEC-03](srs.md#srs-nfr-sec-03)); протокол RESP / TCP 6379.
- **Prometheus** та **Grafana** – збір метрик з /metrics кожні 15 с та сповіщення ([SRS-FUN-16](srs.md#srs-fun-16), [SRS-NFR-OPS-01](srs.md#srs-nfr-ops-01)).
- **Backup Job** (cron + pg_dump) – щоденне резервне копіювання в Amazon S3 ([SRS-NFR-REL-03](srs.md#srs-nfr-rel-03)).

Усі контейнери описані в одному файлі Docker Compose та запускаються від непривілейованого користувача ([SRS-NFR-SEC-05](srs.md#srs-nfr-sec-05)); політика перезапуску та перевірки /healthz забезпечують [SRS-NFR-REL-01](srs.md#srs-nfr-rel-01).

## 2.2 Компонентне моделювання критичного сервісу системи

<a id="diagram-level3"></a>
### 2.2.1 Рівень 3 – Component: Dedicated Server

Як критичний сервіс обрано контейнер Dedicated Server, оскільки він містить основне обчислювальне навантаження (симуляція, ШІ ворогів, реплікація) та всі механізми захисту від маніпуляцій клієнта.

Рисунок 2.3 – Діаграма компонентів Dedicated Server (C4 Level 3)

```mermaid
flowchart TB
    client["<b>Game Client</b><br/>[Container]"]:::container
    api["<b>Progression API</b><br/>[Container: Go]"]:::container
    prom["<b>Prometheus</b><br/>[Container]"]:::container
    eos["<b>EOS JWKS</b><br/>[System_Ext]"]:::ext
    subgraph ds["Dedicated Server [Container: UE5 C++]"]
        subgraph inbound["Inbound"]
            net["<b>NetDriver / RPC Endpoint</b><br/>[Component: APlayerController]<br/>Приймає з'єднання та Server RPC"]:::component
            metrics["<b>MetricsExporter</b><br/>[Component: HTTP /metrics, /healthz]<br/>Tick Rate, frame time, RTT"]:::component
        end
        subgraph guards["Guards / Middleware"]
            login["<b>LoginGuard</b><br/>[Component: AGameMode::PreLogin]<br/>Перевірка EOS ID Token, JWT RS256"]:::component
            rate["<b>RpcRateLimiter</b><br/>[Component: Token Bucket]<br/>Ліміт RPC на з'єднання"]:::component
            valid["<b>ActionValidator</b><br/>[Component: IActionValidator]<br/>Дистанція, LOS, кулдаун, Lag Compensation"]:::component
        end
        subgraph core["Core (бізнес-логіка)"]
            combat["<b>CombatService</b><br/>[Component: UGameInstanceSubsystem]<br/>Серверний розрахунок шкоди з Data Table"]:::component
            match["<b>MatchStateMachine</b><br/>[Component: AGameState + FSM]<br/>Lobby → Wave → Intermission → Finished"]:::component
            spawn["<b>SpawnDirector</b><br/>[Component: UWorldSubsystem]<br/>Хвилі, динамічна складність, EQS"]:::component
            ai["<b>EnemyAI</b><br/>[Component: Behavior Tree, EQS]"]:::component
        end
        subgraph outbound["Outbound / Infrastructure"]
            report["<b>MatchResultReporter</b><br/>[Component: IMatchResultSink]<br/>Формує MatchResult, retry з backoff"]:::component
            httpc["<b>ProgressionApiClient</b><br/>[Component: FHttpModule, async]<br/>REST + Idempotency-Key"]:::component
            jwks["<b>JwksCache</b><br/>[Component: IKeyProvider]<br/>Кеш ключів EOS ≤ 1 год"]:::component
            err["<b>ErrorHandler / Logger</b><br/>[Component: UE_LOG, JSON]<br/>Структуровані логи, коди відмов"]:::component
        end
    end
    client -->|"Server RPC [UE Net / UDP]"| net
    net -->|"PreLogin(token) [виклик]"| login
    login -->|"GetKey(kid) [інтерфейс, DI]"| jwks
    jwks -->|"GET JWKS [HTTPS]"| eos
    net -->|"TryConsume() [виклик]"| rate
    rate -->|"Validate(action) [інтерфейс, DI]"| valid
    valid -->|"ApplyAction() [інтерфейс]"| combat
    combat -->|"OnPlayerDamaged [Observer, Delegate]"| match
    match -->|"OnWaveStart [Observer, Delegate]"| spawn
    spawn -->|"Spawn + Possess [виклик]"| ai
    match -->|"OnMatchFinished [Observer, Delegate]"| report
    report -->|"Send(MatchResult) [інтерфейс, DI]"| httpc
    httpc -->|"POST /v1/matches [REST / HTTPS]"| api
    valid -.->|"Reject(reason)"| err
    rate -.->|"Violation"| err
    login -.->|"AuthFailed"| err
    prom -->|"Scrape [HTTP]"| metrics
    match -.->|"Метрики стану"| metrics
    classDef person fill:#08427b,stroke:#052e56,color:#fff
    classDef system fill:#1168bd,stroke:#0b4884,color:#fff
    classDef container fill:#438dd5,stroke:#2e6295,color:#fff
    classDef component fill:#85bbf0,stroke:#5d82a8,color:#000
    classDef db fill:#438dd5,stroke:#2e6295,color:#fff
    classDef ext fill:#999999,stroke:#6b6b6b,color:#fff
```

Компоненти згруповано за шарами Clean Architecture; залежності спрямовані від зовнішніх шарів до ядра, а ядро звертається до інфраструктури лише через інтерфейси (Dependency Inversion).

**Inbound (вхідний шар):**

- **NetDriver / RPC Endpoint** (APlayerController, Server RPC з атрибутом WithValidation) – приймає з'єднання та серверні RPC; не містить бізнес-логіки, передає запит у шар Guards.
- **MetricsExporter** – вбудований HTTP-ендпоінт /metrics та /healthz ([SRS-FUN-16](srs.md#srs-fun-16), [SRS-FUN-17](srs.md#srs-fun-17)); отримує значення від MatchStateMachine та ігрового циклу.

**Guards / Middleware:**

- **LoginGuard** (перевизначення AGameMode::PreLogin) – перевіряє EOS ID Token до створення PlayerController ([SRS-FUN-06](srs.md#srs-fun-06), [SRS-NFR-SEC-01](srs.md#srs-nfr-sec-01)); ключі отримує через інтерфейс IKeyProvider.
- **RpcRateLimiter** (UActorComponent, алгоритм Token Bucket) – обмежує частоту RPC на з'єднання та ініціює від'єднання при порушеннях ([SRS-FUN-04](srs.md#srs-fun-04), [SRS-NFR-SEC-03](srs.md#srs-nfr-sec-03)).
- **ActionValidator** (інтерфейс IActionValidator) – перевіряє дистанцію, лінію видимості з компенсацією затримки та кулдаун ([SRS-FUN-02](srs.md#srs-fun-02)).

**Core (бізнес-логіка):**

- **CombatService** (UGameInstanceSubsystem) – обчислює шкоду лише на сервері з параметрів Data Table ([SRS-FUN-03](srs.md#srs-fun-03)).
- **MatchStateMachine** (AGameState + скінченний автомат Lobby → Wave → Intermission → Finished) – зберігає авторитетний стан і реплікує його ([SRS-FUN-01](srs.md#srs-fun-01)); публікує події через делегати (патерн Observer).
- **SpawnDirector** (UWorldSubsystem) – генерує хвилі з динамічною складністю ([SRS-FUN-05](srs.md#srs-fun-05)).
- **EnemyAI** (Behavior Tree, EQS) – поведінка ворогів, що виконується лише на сервері.

**Outbound / Infrastructure:**

- **MatchResultReporter** (реалізація IMatchResultSink) – формує MatchResult та повторює відправку з експоненційною затримкою без блокування ігрового циклу ([SRS-FUN-07](srs.md#srs-fun-07), [SRS-NFR-REL-04](srs.md#srs-nfr-rel-04)).
- **ProgressionApiClient** (FHttpModule, асинхронні запити) – REST-клієнт із заголовком Idempotency-Key та сервісним ключем ([SRS-NFR-SEC-02](srs.md#srs-nfr-sec-02)).
- **JwksCache** (реалізація IKeyProvider) – кешує ключі EOS не довше 1 год ([SRS-NFR-SEC-01](srs.md#srs-nfr-sec-01)).
- **ErrorHandler / Logger** – централізована обробка відмов (Reject, Violation, AuthFailed) зі структурованими логами у форматі JSON.

Типи зв'язків: синхронні внутрішньопроцесні виклики (виклик), виклики через інтерфейси з впровадженням залежностей (інтерфейс, DI), події через делегати Unreal Engine (Observer, Delegate). Пунктиром позначено потоки помилок та телеметрії.

## 2.3 Документування архітектурних рішень (Architecture Decision Records)

| ID | Рішення | Статус |
|---|---|---|
| [ADR-001](adr/0001-choice-of-technology.md) | Модель розміщення авторитетного ігрового сервера: Dedicated Server у Docker на AWS EC2 | Approved |

## 2.4 Матриця трасування вимог

Таблиця 2.1 – Трасування вимог SRS в архітектурні елементи

| Вимога | Контейнер (C4 L2) | Компонент (C4 L3) | ADR |
|---|---|---|---|
| [SRS-FUN-01](srs.md#srs-fun-01) | Dedicated Server | MatchStateMachine | [ADR-001](adr/0001-choice-of-technology.md) |
| [SRS-FUN-02](srs.md#srs-fun-02) | Dedicated Server | ActionValidator | [ADR-001](adr/0001-choice-of-technology.md) |
| [SRS-FUN-03](srs.md#srs-fun-03) | Dedicated Server | CombatService | [ADR-001](adr/0001-choice-of-technology.md) |
| [SRS-FUN-04](srs.md#srs-fun-04) | Dedicated Server | RpcRateLimiter | [ADR-001](adr/0001-choice-of-technology.md) |
| [SRS-FUN-05](srs.md#srs-fun-05) | Dedicated Server | SpawnDirector, EnemyAI | – |
| [SRS-FUN-06](srs.md#srs-fun-06) | Dedicated Server | LoginGuard, JwksCache | – |
| [SRS-FUN-07](srs.md#srs-fun-07) | Dedicated Server | MatchResultReporter, ProgressionApiClient | – |
| [SRS-FUN-08](srs.md#srs-fun-08), [SRS-FUN-09](srs.md#srs-fun-09) | Game Client | – | – |
| [SRS-FUN-10](srs.md#srs-fun-10)–[SRS-FUN-14](srs.md#srs-fun-14) | Progression API, PostgreSQL, Redis | – | – |
| [SRS-FUN-15](srs.md#srs-fun-15) | GitHub Actions (зовнішня система) | – | – |
| [SRS-FUN-16](srs.md#srs-fun-16), [SRS-FUN-17](srs.md#srs-fun-17) | Dedicated Server, Prometheus | MetricsExporter | – |
| [SRS-NFR-PERF-01](srs.md#srs-nfr-perf-01), [SRS-NFR-SCAL-01](srs.md#srs-nfr-scal-01) | Dedicated Server | MatchStateMachine, SpawnDirector | [ADR-001](adr/0001-choice-of-technology.md) |
| [SRS-NFR-SEC-02](srs.md#srs-nfr-sec-02), [SRS-NFR-SEC-03](srs.md#srs-nfr-sec-03) | Reverse Proxy, Progression API | ProgressionApiClient, RpcRateLimiter | – |
| [SRS-NFR-REL-01](srs.md#srs-nfr-rel-01) | Dedicated Server (restart policy) | MetricsExporter (/healthz) | [ADR-001](adr/0001-choice-of-technology.md) |
| [SRS-NFR-REL-03](srs.md#srs-nfr-rel-03) | Backup Job | – | – |
