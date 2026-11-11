# Специфікація вимог до програмного забезпечення (SRS)

**Стандарт:** ISO/IEC/IEEE 29148:2018
**Система:** Кооперативна багатокористувацька гра на Unreal Engine 5 з архітектурою Authoritative Dedicated Server
**Версія документа:** 1.0
**Статус:** Approved

## 1 System Context (Контекст системи)

### 1.1 Межі системи (Scope)

До меж системи належать:

- ігровий клієнт на Unreal Engine 5 (Windows);
- контейнеризований Dedicated Server на Unreal Engine 5 (Linux), що виконує авторитетну симуляцію матчу;
- сервіс прогресії (Progression API) на Go з REST API, сховищем PostgreSQL та кешем Redis;
- конвеєр CI/CD у GitHub Actions та контур спостережуваності (Prometheus, Grafana).

Поза межами системи перебувають: ідентифікація користувачів, лобі та підбір гравців (Epic Online Services), хостинг обчислювальних ресурсів (AWS EC2), зберігання контейнерних образів (GitHub Container Registry), зберігання резервних копій (Amazon S3).

### 1.2 Взаємодія із зовнішнім середовищем

| Зовнішній учасник | Тип | Взаємодія із системою |
|---|---|---|
| Гравець | Person | Керує ігровим клієнтом; ініціює лобі, приєднується до матчу |
| Оператор (DevOps) | Person | Ініціює розгортання, переглядає дашборди Grafana, отримує сповіщення |
| Epic Online Services | System_Ext | Автентифікація (Connect Interface, ID Token), лобі (Lobbies), сесії; HTTPS |
| AWS EC2 | System_Ext | Хост контейнерів Dedicated Server та сервісів |
| GitHub Container Registry | System_Ext | Зберігання Docker-образів; HTTPS (OCI Distribution API) |
| Amazon S3 | System_Ext | Зберігання резервних копій PostgreSQL; HTTPS |

### 1.3 Обмеження та припущення

- Кількість гравців у сесії: від 2 до 3.
- Розгортання: один регіон AWS (eu-central-1), Docker Compose на інстансах EC2; оркестрація Kubernetes не використовується.
- Dedicated Server збирається з вихідного коду Unreal Engine 5 (Source Build).

## 2 Functional Requirements (Функціональні вимоги)

### 2.1 Модуль ігрового сервера (Dedicated Server)

<a id="srs-fun-01"></a>**SRS-FUN-01.** Dedicated Server повинен реплікувати клієнтам авторитетний стан матчу (позиції персонажів, здоров'я, боєприпаси, номер хвилі, кількість ворогів) з частотою не нижче частоти оновлення сервера, визначеної в SRS-NFR-PERF-01.

<a id="srs-fun-02"></a>**SRS-FUN-02.** Dedicated Server повинен відхиляти серверний RPC пострілу, якщо дистанція до цілі перевищує дальність зброї, ціль поза лінією видимості з урахуванням компенсації затримки або кулдаун зброї не завершено.

<a id="srs-fun-03"></a>**SRS-FUN-03.** Dedicated Server повинен обчислювати шкоду виключно на сервері на основі параметрів зброї з таблиці даних (Data Table), ігноруючи значення шкоди, передані клієнтом.

<a id="srs-fun-04"></a>**SRS-FUN-04.** Dedicated Server повинен обмежувати частоту серверних RPC кожного з'єднання алгоритмом Token Bucket відповідно до SRS-NFR-SEC-03 та від'єднувати клієнта після 3 перевищень ліміту протягом 10 с.

<a id="srs-fun-05"></a>**SRS-FUN-05.** Dedicated Server повинен генерувати хвилі ворогів серверним компонентом Spawn Director, обчислюючи кількість ворогів хвилі за формулою, що враховує номер хвилі, кількість живих гравців та їхнє середнє здоров'я.

<a id="srs-fun-06"></a>**SRS-FUN-06.** Dedicated Server повинен приймати з'єднання лише від клієнтів, що передали дійсний EOS ID Token, перевірений згідно з SRS-NFR-SEC-01.

<a id="srs-fun-07"></a>**SRS-FUN-07.** Після завершення матчу Dedicated Server повинен формувати об'єкт MatchResult та надсилати його до Progression API із заголовком Idempotency-Key, що дорівнює ідентифікатору матчу.

### 2.2 Модуль сесій

<a id="srs-fun-08"></a>**SRS-FUN-08.** Ігровий клієнт повинен створювати лобі в EOS Lobbies з атрибутами версії збірки та максимальної кількості учасників (3).

<a id="srs-fun-09"></a>**SRS-FUN-09.** Ігровий клієнт повинен приєднуватися до Dedicated Server за адресою, опублікованою в атрибуті лобі, лише за збігу версії збірки клієнта та сервера.

### 2.3 Модуль сервісу прогресії (Progression API)

<a id="srs-fun-10"></a>**SRS-FUN-10.** Progression API повинен зберігати MatchResult та нараховувати досвід і нагороди учасникам матчу в межах однієї транзакції PostgreSQL.

<a id="srs-fun-11"></a>**SRS-FUN-11.** Progression API повинен повертати збережений результат без повторного запису, якщо запит містить Idempotency-Key, що вже оброблений протягом останніх 24 год.

<a id="srs-fun-12"></a>**SRS-FUN-12.** Progression API повинен надавати ендпоінт читання профілю гравця (GET /v1/players/me) з даними рівня, досвіду, статистики та інвентарю.

<a id="srs-fun-13"></a>**SRS-FUN-13.** Progression API повинен надавати доступ до профілю лише власнику профілю, ідентифікованому за полем sub перевіреного JWT (захист від BOLA).

<a id="srs-fun-14"></a>**SRS-FUN-14.** Progression API повинен надавати ендпоінт видалення профілю (DELETE /v1/players/me), що видаляє всі персональні дані гравця.

### 2.4 Модуль операційного контуру

<a id="srs-fun-15"></a>**SRS-FUN-15.** Конвеєр CI повинен для кожного Pull Request збирати Docker-образ Dedicated Server та запускати Unreal Automation Tests, SonarQube Quality Gate та сканування Trivy.

<a id="srs-fun-16"></a>**SRS-FUN-16.** Dedicated Server повинен експортувати метрики server_tick_rate, server_frame_time_ms, connected_players, player_rtt_ms, process_memory_bytes на HTTP-ендпоінті /metrics у форматі Prometheus Exposition Format.

<a id="srs-fun-17"></a>**SRS-FUN-17.** Dedicated Server повинен надавати ендпоінт /healthz, що повертає HTTP 200 за умови активного ігрового циклу (останній тик не старший за 1 с).

## 3 Non-Functional Requirements (Нефункціональні вимоги)

| ID | Категорія | Вимога |
|---|---|---|
| <a id="srs-nfr-perf-01"></a>SRS-NFR-PERF-01 | Продуктивність | Частота оновлення Dedicated Server повинна становити ≥ 30 Гц; p95 тривалості кадру ≤ 33 мс при 3 гравцях і 40 активних ворогах |
| <a id="srs-nfr-perf-02"></a>SRS-NFR-PERF-02 | Продуктивність | p95 RTT клієнт–сервер повинен становити ≤ 100 мс для клієнтів у регіоні eu-central-1 |
| <a id="srs-nfr-perf-03"></a>SRS-NFR-PERF-03 | Продуктивність | Progression API повинен забезпечувати p95 ≤ 150 мс та p99 ≤ 300 мс при 100 RPS протягом 5 хв; частка відповідей 5xx < 1 % |
| <a id="srs-nfr-scal-01"></a>SRS-NFR-SCAL-01 | Масштабованість | Інстанс c6i.large повинен обслуговувати ≥ 4 одночасні сесії по 3 гравці з дотриманням SRS-NFR-PERF-01 |
| <a id="srs-nfr-scal-02"></a>SRS-NFR-SCAL-02 | Масштабованість | Частка влучань у кеш Redis для GET /v1/players/me повинна становити ≥ 80 % при навантаженні SRS-NFR-PERF-03 |
| <a id="srs-nfr-sec-01"></a>SRS-NFR-SEC-01 | Безпека | JWT повинен перевірятися за підписом RS256 (ключі JWKS EOS із кешуванням ≤ 1 год), полями exp, iss та aud |
| <a id="srs-nfr-sec-02"></a>SRS-NFR-SEC-02 | Безпека | Progression API повинен приймати лише TLS 1.2+; виклики Dedicated Server → API повинні автентифікуватися сервісним ключем зі сховища AWS SSM Parameter Store |
| <a id="srs-nfr-sec-03"></a>SRS-NFR-SEC-03 | Безпека | Ліміт частоти: ≤ 20 RPC пострілу/с та ≤ 60 RPC руху/с на з'єднання; ≤ 60 запитів/хв на гравця до Progression API (HTTP 429) |
| <a id="srs-nfr-sec-04"></a>SRS-NFR-SEC-04 | Безпека | Docker-образи повинні містити 0 вразливостей рівня Critical та High за Trivy; SonarQube Quality Gate – 0 Blocker і 0 Critical зауважень |
| <a id="srs-nfr-sec-05"></a>SRS-NFR-SEC-05 | Безпека | Контейнери повинні запускатися від непривілейованого користувача (UID ≠ 0) з файловою системою лише для читання, крім каталогу логів |
| <a id="srs-nfr-comp-01"></a>SRS-NFR-COMP-01 | Комплаєнс (GDPR) | Система повинна зберігати лише EOS Product User ID та нікнейм; видалення за SRS-FUN-14 повинно виконуватися ≤ 30 діб, включно з резервними копіями |
| <a id="srs-nfr-rel-01"></a>SRS-NFR-REL-01 | Надійність | Контейнер Dedicated Server повинен бути перезапущений ≤ 30 с після 3 невдалих перевірок /healthz з інтервалом 10 с |
| <a id="srs-nfr-rel-02"></a>SRS-NFR-REL-02 | Надійність | Повторна відправка MatchResult з тим самим Idempotency-Key повинна створювати 0 дублікатів |
| <a id="srs-nfr-rel-03"></a>SRS-NFR-REL-03 | Надійність | Резервна копія PostgreSQL повинна створюватися щодоби в Amazon S3; RPO ≤ 24 год, RTO ≤ 1 год |
| <a id="srs-nfr-rel-04"></a>SRS-NFR-REL-04 | Надійність | При недоступності Progression API Dedicated Server повинен повторювати відправку MatchResult з експоненційною затримкою (5 спроб, 1–16 с) без блокування ігрового циклу |
| <a id="srs-nfr-qual-01"></a>SRS-NFR-QUAL-01 | Якість коду | Покриття тестами коду Progression API повинно становити ≥ 60 % рядків; модулі валідації RPC та Spawn Director повинні мати Unreal Automation Tests |
| <a id="srs-nfr-ops-01"></a>SRS-NFR-OPS-01 | Спостережуваність | Prometheus повинен збирати метрики з інтервалом 15 с; Grafana Alerting повинна сповіщати оператора при server_tick_rate < 25 Гц понад 60 с |
| <a id="srs-nfr-inf-01"></a>SRS-NFR-INF-01 | Інфраструктура | Розмір образу Dedicated Server повинен становити ≤ 2 ГБ; тривалість конвеєра CI з кешем збірки – ≤ 45 хв |

## 4 Verification Criteria (Критерії верифікації)

| Вимога | Метод | Інструмент та критерій проходження |
|---|---|---|
| SRS-FUN-01, SRS-FUN-03 | Test | Unreal Automation Tests (Functional Tests) у режимі PIE з 2 клієнтами та Dedicated Server: стан на клієнтах збігається із сервером після 300 тиків |
| SRS-FUN-02, SRS-FUN-04 | Test | Unreal Automation Tests: модифікований тестовий клієнт надсилає недопустимі RPC; очікується 100 % відхилень і від'єднання за SRS-FUN-04 |
| SRS-FUN-05 | Test | Unit-тести Spawn Director (Unreal Automation Framework) для таблиці вхідних параметрів із граничними значеннями |
| SRS-FUN-06, SRS-NFR-SEC-01 | Test | Тести з простроченим, непідписаним та чужим (aud) JWT: очікується відмова у з'єднанні / HTTP 401 |
| SRS-FUN-07, SRS-FUN-10, SRS-FUN-11, SRS-NFR-REL-02 | Test | Go test (інтеграційні тести з Testcontainers PostgreSQL): 10 паралельних повторів з одним Idempotency-Key створюють 1 запис |
| SRS-FUN-12, SRS-FUN-13, SRS-FUN-14 | Test | Postman/Newman колекція в CI: доступ до чужого профілю повертає HTTP 403; після DELETE профіль повертає HTTP 404 |
| SRS-FUN-15, SRS-NFR-SEC-04 | Inspection | Журнал GitHub Actions: етапи SonarQube Quality Gate та Trivy (--severity CRITICAL,HIGH --exit-code 1) завершуються успішно |
| SRS-FUN-16, SRS-FUN-17, SRS-NFR-OPS-01 | Demonstration | Prometheus Targets у стані UP; дашборд Grafana відображає метрики; тестове правило сповіщення спрацьовує |
| SRS-NFR-PERF-01, SRS-NFR-SCAL-01 | Test | Навантажувальна симуляція Python-ботами: 4 сесії × 3 боти протягом 15 хв; метрика server_frame_time_ms p95 ≤ 33 мс у Prometheus |
| SRS-NFR-PERF-02 | Analysis | Аналіз метрики player_rtt_ms (p95) за 7 діб тестової експлуатації |
| SRS-NFR-PERF-03, SRS-NFR-SCAL-02, SRS-NFR-SEC-03 | Test | k6: сценарії Constant Arrival Rate (100 RPS, 5 хв) та Spike (0 → 300 RPS); пороги http_req_duration p(95)<150, http_req_failed<0.01; перевірка HTTP 429 |
| SRS-NFR-SEC-02 | Test | testssl.sh для ендпоінта API: TLS 1.0/1.1 відхиляються |
| SRS-NFR-SEC-05, SRS-NFR-INF-01 | Inspection | docker inspect (User, ReadonlyRootfs), docker image ls (розмір); Trivy config scan Dockerfile |
| SRS-NFR-REL-01 | Test | Хаос-тест: зупинка ігрового циклу (SIGSTOP); фіксація часу перезапуску контейнера ≤ 30 с |
| SRS-NFR-REL-03 | Demonstration | Відновлення резервної копії з S3 у тестовий екземпляр PostgreSQL з фіксацією часу |
| SRS-NFR-REL-04 | Test | Інтеграційний тест із вимкненим Progression API: 5 спроб у журналі, Tick Rate ≥ 30 Гц |
| SRS-NFR-QUAL-01 | Analysis | go test -cover та звіт SonarQube Coverage |
| SRS-NFR-COMP-01 | Inspection | Рев'ю схеми БД (DDL) та процедури ротації резервних копій |

## 5 Glossary (Глосарій)

| Термін | Визначення |
|---|---|
| Authoritative Dedicated Server | Окремий серверний процес без рендерингу, що є єдиним джерелом істини щодо стану матчу; клієнти лише надсилають запити на дії |
| Replication | Механізм Unreal Engine для передачі змін властивостей акторів від сервера до клієнтів з урахуванням релевантності та пріоритету |
| RPC (Remote Procedure Call) | Виклик функції на віддаленій стороні; Server RPC виконується на сервері за ініціативою клієнта |
| Tick Rate | Кількість ітерацій ігрового циклу сервера за секунду (Гц) |
| Latency / RTT | Затримка передачі даних; RTT – час проходження пакета клієнт → сервер → клієнт |
| Idempotency-Key | Унікальний ключ запиту, що гарантує однаковий результат при повторній обробці того самого запиту |
| JWT (JSON Web Token) | Підписаний токен з набором тверджень (claims), що використовується для автентифікації; EOS ID Token є JWT |
| Token Bucket | Алгоритм обмеження частоти: запит обробляється за наявності токена в «кошику», що поповнюється з фіксованою швидкістю |
| Spawn Director | Серверний компонент, що визначає момент, місце та кількість появи ворогів |
| BOLA | Broken Object Level Authorization – вразливість доступу до чужих об'єктів (OWASP API1:2023) |

## 6 Трасування вимог

Відповідність вимог архітектурним елементам наведено у файлі [2_design.md](2_design.md); обґрунтування вибору моделі розміщення сервера – у [ADR-001](adr/0001-choice-of-technology.md).
