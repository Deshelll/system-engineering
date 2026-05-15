# 4. Architecture Decisions (ADRs)

## 4.1. Назначение раздела

ADR (Architecture Decision Records) фиксируют ключевые архитектурные решения с обоснованием, рассмотренными альтернативами и последствиями. ADR — это не «инструкция», а **исторический документ**: через год команда сможет понять, _почему_ выбрали именно эту технологию, и осознанно пересмотреть решение, если условия изменятся.

**Формат:** упрощённый MADR.

**Общий контекст для всех ADR:**

-   Целевая среда: Kubernetes (cloud-agnostic, предпочтение AWS/GCP)
-   Масштаб: 10-50 продуктовых команд, 100-500 сервисов
-   Подход: build (self-hosted), strong preference for open source
-   Заказчик: greenfield с возможным легаси-Prometheus

## 4.2. ADR-001: Metrics Backend
**ADR-001: Выбор metrics backend**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-ING-001, FR-STO-001..003, FR-QRY-001,
                      QAS-PERF-01..03, QAS-SCAL-01..02, QAS-AVAIL-02,
                      C-TECH-02 (open source), C-COST-01

### Контекст

Метрики — основа alerting и SLO в СМА. Backend должен принимать ~1M samples/sec на старте с прогнозом роста до 10M, поддерживать long retention (13 месяцев), горизонтальное масштабирование, multi-tenancy и tiered storage. PromQL — обязательный contract (FR-QRY-001).

Стандартный Prometheus single-binary не подходит: его архитектура (single-node, локальный TSDB) ограничивает масштабирование и не поддерживает multi-tenancy. Нужно horizontally scalable решение, сохраняющее PromQL-совместимость.

### Рассмотренные варианты

**Вариант 1: Prometheus + Thanos**

Преимущества:

-   Самая зрелая комбинация для long-term storage поверх Prometheus
-   Архитектура «sidecar + object storage» проста для понимания
-   Сильное community, много production-кейсов
-   100% совместимость с PromQL

Недостатки:

-   Operational complexity: 5+ компонентов (sidecar, store, compactor, query, ruler)
-   Query performance на больших окнах хуже, чем у Mimir/VictoriaMetrics
-   Multi-tenancy — bolt-on, не first-class

**Вариант 2: Grafana Mimir**

Преимущества:

-   Built-in multi-tenancy (first-class) — критично для нашего use case
-   Horizontal scalability проектировался с нуля под cloud-native
-   100% совместимость с PromQL + дополнительные оптимизации (query sharding)
-   Active development под Grafana Labs
-   Sub-second queries даже на годовых окнах (per-block parallelism)
-   Helm chart и operator official

Недостатки:

-   Молодой (2022) — меньше production-кейсов, чем у Thanos
-   ~20 микросервисов в полной топологии (но есть monolithic mode)
-   Apache 2.0, но development доминирует Grafana Labs

**Вариант 3: VictoriaMetrics (cluster)**

Преимущества:

-   Лучшая performance/cost ratio по независимым benchmarks
-   Минимальный resource footprint (~3-5x меньше памяти, чем Prometheus)
-   Простая архитектура (3 компонента: vminsert, vmstorage, vmselect)
-   MetricsQL — superset PromQL
-   Multi-tenancy в cluster edition

Недостатки:

-   MetricsQL не 100% совместим с PromQL: есть отличия в edge cases (sub-queries, некоторые функции)
-   Меньше интеграций с CNCF-экосистемой
-   Решения по open source vs enterprise split неоднозначные (часть features — в paid)
-   Object storage tier появился позже и менее зрел, чем у Mimir/Thanos

**Вариант 4: Cortex**

Преимущества:

-   Историческая база Mimir, проверенная Grafana Cloud и AWS Managed Prometheus

Недостатки:

-   Development замедлился после форка Mimir
-   Сообщество мигрирует на Mimir
-   **Эффективно legacy**

### Решение

Выбран **Grafana Mimir**.

Обоснование:

-   **Multi-tenancy как first-class concept**  — решающий фактор. В Thanos и VictoriaMetrics multi-tenancy — bolt-on, в Mimir — основа архитектуры. Это напрямую отвечает QAS-SCAL-01 (изоляция тенантов) и FR-ING-006.
-   **PromQL 100%**  — без сюрпризов миграции с legacy Prometheus и совместимость со всем существующим tooling (rules, dashboards, alerts).
-   **Query sharding**  — критичен для QAS-PERF-03 на больших окнах (long-range queries для capacity planning, S7).
-   **Operational maturity**  — Helm chart + operator + monolithic mode на старте, microservices mode при росте — даёт incremental complexity.

Cortex отвергнут как effectively legacy. Thanos — хорошая альтернатива, но multi-tenancy слабее. VictoriaMetrics — самый сильный challenger по cost, но MetricsQL ≠ PromQL создаёт риск vendor lock-in на уровне query language.

### Последствия

Положительные:

-   Native multi-tenancy упрощает реализацию FR-ING-006 и QAS-SEC-01
-   Sub-second long-range queries без специальных оптимизаций
-   Helm-based deploy ускоряет initial rollout

Отрицательные:

-   Operational complexity выше, чем у VictoriaMetrics (~20 микросервисов в production mode)
-   Зависимость от roadmap Grafana Labs (хотя проект Apache 2.0)
-   Стоимость cardinality в Mimir выше, чем в VictoriaMetrics (требует строгих квот — FR-ADM-002)

Нейтральные:

-   Требуется S3-compatible object storage как hard dependency (см. ADR-009)
-   Команда должна освоить специфику Mimir (block storage, compactor tuning)

### Связи

ADR-002 (logs), ADR-005 (alerting), ADR-009 (storage strategy), FR-ING-006, QAS-SCAL-01.

## 4.3. ADR-002: Logs Backend
**ADR-002: Выбор logs backend**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-ING-002, FR-STO-001..003, FR-QRY-002, FR-QRY-003, QAS-PERF-01..03, QAS-SCAL-01, C-COST-02

### Контекст

Логи — самый «дорогой» сигнал observability: высокий volume, переменная structure, expensive full-text search. Нужно принимать ~500K log entries/sec с burst до 10x, обеспечивать query latency < 5 сек на 1ч окне (QAS-PERF-03b), поддерживать корреляцию с метриками и трейсами (FR-QRY-003), retention 30 дней hot + 90 warm.

Главный архитектурный выбор — между двумя парадигмами:

-   **Indexed search**  (Elasticsearch-family): индексируется всё, быстрый full-text search, но дорого по storage и compute
-   **Label-based indexing**  (Loki): индексируются только labels, content — в object storage; дёшево, но full-text search дороже

### Рассмотренные варианты

**Вариант 1: Grafana Loki**

Преимущества:

-   В 10-20x раз дешевле Elasticsearch по storage (по бенчмаркам Grafana)
-   Label-based model совместима с Prometheus mental model — снижает cognitive load
-   LogQL похож на PromQL — нативная корреляция метрик↔логов (FR-QRY-003)
-   Object storage как primary tier — нативный tiering
-   Multi-tenancy first-class (как Mimir)

Недостатки:

-   Full-text search на большом окне медленнее, чем у inverted-index решений
-   Не подходит для use case «найти иголку в стоге сена» без labels
-   Требует дисциплины labeling от продуктовых команд

**Вариант 2: OpenSearch (форк Elasticsearch)**

Преимущества:

-   Лучший full-text search в индустрии, inverted index
-   Сильная экосистема (Kibana, плагины)
-   Apache 2.0 (в отличие от Elasticsearch SSPL)
-   Подходит для security/audit логов с произвольным поиском

Недостатки:

-   Storage cost в 10x раз выше Loki
-   Высокий memory footprint (heap-tuning — отдельная дисциплина)
-   Multi-tenancy — bolt-on (через index templates / OpenSearch Security)
-   Корреляция с метриками — через UI hops, не native
-   Operational complexity (shard management, rolling restart pain)

**Вариант 3: ClickHouse**

Преимущества:

-   Excellent performance/cost для structured logs
-   SQL — знакомый язык, мощные агрегации
-   Используется в production у крупных игроков (Cloudflare, Uber)
-   Может объединить logs + traces (см. ADR-003)

Недостатки:

-   Нет специфичного для логов tooling (Kibana-equivalent)
-   LogQL/PromQL совместимость отсутствует — кастомные дашборды
-   Schema design нетривиален; миграция schema болезненна
-   Multi-tenancy через databases / row policies — менее зрелое
-   Команде нужны ClickHouse-специалисты

**Вариант 4: Elasticsearch (proprietary)**

Недостатки:

-   SSPL лицензия — нарушает C-TECH-02 (preference for open source)
-   **Отклонён без дальнейшего рассмотрения**

### Решение

Выбран **Grafana Loki** как primary log backend.

Обоснование:

-   **Cost**  — главный driver для логов (C-COST-02). Loki в 10x дешевле OpenSearch при сопоставимом UX для типового use case «логи известного сервиса за известное окно».
-   **Корреляция metrics↔logs↔traces**  (FR-QRY-003) — native через label compatibility с Prometheus/Mimir и Grafana UI. У OpenSearch это hops через UI.
-   **Multi-tenancy first-class**  — консистентно с ADR-001.
-   **Mental model alignment**  — для продуктовых команд один и тот же labeling model для метрик и логов снижает cognitive load и ускоряет onboarding (S1).

OpenSearch отвергнут по cost и operational reasons, но **зарезервирован как secondary** для специфичных use cases (audit logs, security logs с произвольным full-text search).

ClickHouse — сильный кандидат, особенно если объединить с traces (ADR-003), но переход на SQL ломает unified mental model и требует больше custom tooling.

### Последствия

Положительные:

-   Storage cost в пределах C-COST-02
-   Native unified UI с метриками и трейсами через Grafana (см. ADR-006)
-   Консистентная operational model с Mimir

Отрицательные:

-   Продуктовые команды должны дисциплинированно использовать labels (требует guidelines и review в onboarding S1)
-   Полнотекстовый поиск без labels — медленный; этот use case нужно явно ограничить
-   Возможно, потребуется secondary OpenSearch cluster для security/audit logs — увеличивает общую сложность

Нейтральные:

-   Те же зависимости, что у Mimir: S3-compatible storage, Helm/operator deploy

### Связи

ADR-001 (metrics), ADR-003 (traces), ADR-009 (storage), FR-QRY-003.

## 4.4. ADR-003: Traces Backend

**ADR-003: Выбор traces backend**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-ING-003, FR-STO-001, FR-QRY-003,
                      QAS-PERF-01, QAS-PERF-03c, C-COST-01

### Контекст

Трейсы — самый «горячий» сигнал по volume на span, но retention короче (7 дней по FR-STO-001). Главный use case — расследование (S3): найти trace по trace_id, посмотреть waterfall, перейти к логам и метрикам соответствующих spans.

Высокий volume трейсов требует sampling strategy (head-based в OTel Collector, либо tail-based). Backend должен это поддерживать.

### Рассмотренные варианты

**Вариант 1: Grafana Tempo**

Преимущества:

-   Object storage primary — radically cheaper, чем index-heavy решения
-   Search by trace_id — мгновенный (главный use case)
-   TraceQL — растущий feature set для search
-   Native интеграция с Loki/Mimir/Grafana — FR-QRY-003 «из коробки»
-   Multi-tenancy first-class
-   Same operational mental model, что Mimir и Loki

Недостатки:

-   TraceQL search по attributes медленнее, чем у Jaeger/Elasticsearch
-   Молодой проект (2020) — меньше production кейсов

**Вариант 2: Jaeger**

Преимущества:

-   Зрелый CNCF проект
-   Хороший UI для trace exploration
-   Различные backends (Elasticsearch, Cassandra, ClickHouse через jaeger-clickhouse)

Недостатки:

-   Storage cost высокий (через Elasticsearch backend)
-   Корреляция с метриками/логами слабее, чем Tempo+Grafana
-   Multi-tenancy слабая
-   Project momentum снижается на фоне OpenTelemetry/Tempo

**Вариант 3: ClickHouse (через SigNoz-style или custom)**

Преимущества:

-   Очень дешёвое хранение
-   Мощный SQL-based search по attributes (лучше TraceQL для сложных запросов)
-   Возможность объединить с logs (ADR-002 рассматривал ClickHouse)

Недостатки:

-   Custom schema management
-   Корреляция через custom UI или Grafana plugins
-   Нет established «traces backend» статуса — больше DIY

### Решение

Выбран **Grafana Tempo**.

Обоснование:

-   **Главный use case для on-call — search by trace_id**, который мы получаем из логов и алертов. Tempo оптимизирован под него (мгновенный lookup).
-   **Cost**  — критично для traces из-за volume. Object-storage-primary архитектура Tempo лучшая в классе.
-   **Stack consistency**  — Mimir+Loki+Tempo как единый Grafana stack даёт огромную экономию на operational overhead и unified UX (FR-QRY-003, ADR-006).
-   **TraceQL**  растёт, и для сложного search по attributes можно добавить дополнительные инструменты (например, SigNoz/Jaeger для специфичных нужд).

### Последствия

Положительные:

-   Cost-effective storage для high-volume traces
-   Native корреляция metrics↔logs↔traces через Grafana
-   Унифицированная operational model с Mimir/Loki

Отрицательные:

-   Сложный attribute-based search медленнее, чем у Elasticsearch-based решений
-   Необходимость sampling strategy в OTel Collector (head-based на старте, tail-based — следующий шаг)

Нейтральные:

-   Sampling — отдельная архитектурная сабтема, требует guidelines для команд

### Связи

ADR-001, ADR-002, ADR-004 (OTel Collector), ADR-006, FR-QRY-003.

## 4.5. ADR-004: Telemetry Collection Layer

**ADR-004: Выбор telemetry collection (agents/collectors)**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-ING-001..005, C-TECH-03 (OTel mandatory),
                      QAS-PERF-02 (backpressure)
### Контекст

Collection layer — это «вход» в платформу: то, что собирает метрики/логи/трейсы из приложений, инфраструктуры и K8s и отправляет в backends. Это **самая видимая для продуктовых команд** часть СМА (через SDK и agents), поэтому выбор сильно влияет на UX onboarding (S1).

Constraint C-TECH-03 требует OpenTelemetry как primary стандарт.

### Рассмотренные варианты

**Вариант 1: OpenTelemetry Collector (всё через него)**

Преимущества:

-   Единый стандарт для всех типов сигналов
-   Vendor-neutral — критично для долгосрочности (C-TECH-02)
-   Богатый набор processors (filtering, batching, redaction, sampling)
-   Mode flexibility: DaemonSet (agent), Deployment (gateway), либо hybrid
-   Active development под CNCF

Недостатки:

-   Memory footprint выше, чем у specialized agents (Fluent Bit для логов)
-   Не самый эффективный сборщик логов по cost (per-log-line overhead)
-   Конфигурация YAML может становиться громоздкой

**Вариант 2: Vendor-specific agents (Prometheus + Promtail + Tempo SDK)**

Преимущества:

-   Каждый agent оптимизирован под свой backend
-   Promtail — особенно эффективен для Loki (минимальный footprint)
-   Меньше координации между командами и operations

Недостатки:

-   3 agents вместо 1 — больше operational overhead
-   Конфигурация не унифицирована — больше mental load
-   Vendor lock-in на уровне SDK — нарушает C-TECH-03

**Вариант 3: Hybrid — Fluent Bit для логов + OTel Collector для метрик/трейсов**

Преимущества:

-   Fluent Bit — лучший в классе для log collection (low overhead, mature)
-   OTel Collector для metrics/traces

Недостатки:

-   Два разных tools — двойная конфигурация и operational overhead
-   Для backpressure (QAS-PERF-02) нужны разные подходы в двух tools

**Вариант 4: Vector (Datadog) + OTel**

Преимущества:

-   Vector — Rust-based, очень эффективный

Недостатки:

-   Меньше ecosystem
-   Дополнительный инструмент сверх OTel

### Решение

Выбран **OpenTelemetry Collector с топологией agent + gateway**:

-   **Agent (DaemonSet)**  на каждой ноде — собирает локальную телеметрию (node metrics, K8s logs, app traces/metrics через OTLP)
-   **Gateway (Deployment)**  — централизованная обработка перед отправкой в backends: tenant routing, redaction (FR-ING-006, QAS-SEC-02), tail-based sampling, backpressure (QAS-PERF-02)

Обоснование:

-   **C-TECH-03**  делает OTel обязательным как минимум для метрик/трейсов
-   Использовать OTel и для логов (хоть это менее эффективно) даёт  **unified collection layer**  — один инструмент, одна конфигурация, один operational model
-   **Agent+Gateway топология**  даёт лучшее из обоих миров: локальный buffering + централизованная политика
-   Gateway — естественное место для  **per-tenant rate limiting**  (QAS-SCAL-01) и  **PII redaction**  (QAS-SEC-02)

Hybrid (Fluent Bit + OTel) рассматривался, но операционная сложность двух стеков перевешивает marginal cost-efficiency для логов на нашем масштабе.

### Последствия

Положительные:

-   Unified observability pipeline — одна конфигурация для всех signals
-   Gateway как естественная point of control для security и multi-tenancy
-   Vendor-neutral SDK для приложений (OTel SDK для всех языков)

Отрицательные:

-   Higher resource footprint, чем у specialized agents (особенно для логов)
-   При очень высокой нагрузке логов, возможно, потребуется добавить Fluent Bit рядом (но как exception, не как rule)

Нейтральные:

-   Gateway становится критичным компонентом — нужен HA deployment и self-monitoring (QAS-MON-01)
-   Команды должны использовать OTel SDK, а не legacy Prometheus client (миграционная боль для legacy сервисов)

### Связи

ADR-001..003 (backends), ADR-007 (multi-tenancy enforcement в gateway), QAS-PERF-02, QAS-SEC-02.

## 4.6. ADR-005: Alerting & Notification Stack

**ADR-005: Выбор alerting stack**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-RUL-001..005, FR-NTF-001..005, FR-SIL-001..003, FR-SLO-001..003, QAS-PERF-01, QAS-AVAIL-01

### Контекст

Alerting — критичный компонент: его availability — самая высокая среди функций (QAS-AVAIL-01). Нужны: декларативные правила (FR-RUL-001), multi-condition + multi-burn-rate (FR-RUL-002/003), routing/grouping (FR-RUL-004, FR-NTF-001), silence management (FR-SIL), SLO/error budget (FR-SLO).

Также важна **граница ответственности** между alerting и notification routing:

-   _Alerting_  — генерация алертов из правил (Prometheus-style)
-   _Notification routing_  — маршрутизация, эскалация, ack (Alertmanager-style)
-   _On-call response_  — расписания, mobile, voice (см. ADR-008)

### Рассмотренные варианты

**Вариант 1: Prometheus Alertmanager + Mimir Ruler**

Преимущества:

-   Mimir Ruler выполняет PromQL-правила нативно — нулевая friction с metrics backend (ADR-001)
-   Alertmanager — де-факто стандарт routing/grouping/silence
-   Multi-tenancy в обоих (Mimir Ruler, Alertmanager в Mimir distribution)
-   Зрелые, проверенные production-deployment'ом тысяч компаний

Недостатки:

-   SLO/error budget — не из коробки; нужен Sloth или Pyrra для генерации правил
-   UI для alerts/silences базовый (но интегрируется с Grafana)
-   Notification channels базовые — для voice/SMS нужен external service (см. ADR-008)

**Вариант 2: Grafana Alerting (unified)**

Преимущества:

-   Unified UI alerting в Grafana — лучший UX
-   Поддержка multi-datasource правил (alert на joined metrics+logs)
-   Built-in silence/routing UI
-   Provisioning через файлы / API / Terraform

Недостатки:

-   Grafana Alerting хранит state — нужна HA конфигурация Grafana (что осложняет deployment)
-   Multi-tenancy слабее, чем у Mimir Ruler + Alertmanager
-   Vendor coupling: alerting привязан к Grafana как платформе
-   Менее зрелый на масштабе тысяч правил (исторически Alertmanager оптимизирован под scale)

**Вариант 3: Custom build (Kafka + custom evaluator + Alertmanager)**

Преимущества:

-   Максимальная гибкость

Недостатки:

-   Огромная стоимость разработки и поддержки

**Вариант 4: VictoriaMetrics vmalert + Alertmanager**

Преимущества:

-   Эффективнее, чем Mimir Ruler по ресурсам

Недостатки:

-   Требует VictoriaMetrics как metrics backend (мы выбрали Mimir, ADR-001)
-   **Несовместимо с ADR-001**

### Решение

Выбран **Mimir Ruler + Prometheus Alertmanager** (Alertmanager поставляется как часть Mimir distribution в HA-режиме).

Для SLO-правил используется **Sloth** как декларативный generator: разработчик описывает SLO в YAML, Sloth генерирует multi-burn-rate alerting rules (FR-SLO-003).

Обоснование:

-   **Native integration с Mimir**  (ADR-001) — нулевые потери при выполнении PromQL правил, multi-tenancy консистентна
-   **Alertmanager — proven at scale**: routing tree, grouping, inhibition, silence — всё работает из коробки (FR-NTF-001..004, FR-SIL-001..003)
-   **Декларативная конфигурация через YAML/CRD**  — соответствует FR-RUL-001 и dashboards-as-code mindset (FR-UI-002)
-   **Sloth решает SLO use case**  без custom разработки и без vendor lock-in (Apache 2.0)
-   Grafana Alerting рассматривается как  **complementary UI**  — поверх Mimir Alertmanager для visualization и interactive silence management (см. ADR-006)

### Последствия

Положительные:

-   Полная PromQL экспрессивность для правил (FR-RUL-002)
-   HA Alertmanager — критично для QAS-AVAIL-01 (alerting должен пережить отказы)
-   SLO-as-code через Sloth — низкий barrier для adoption
-   Стандартные интеграции с PagerDuty/Slack/etc. через Alertmanager receivers

Отрицательные:

-   Два UI для alerting: Mimir/Alertmanager API + Grafana UI (нужно явно документировать роли)
-   Sloth — отдельный компонент; нужно версионировать и сопровождать
-   Multi-condition (joined metrics+logs alerting) — не из коробки; для таких случаев нужно использовать Grafana Alerting как exception

Нейтральные:

-   Notification к voice/SMS/escalation — отдельный stack (ADR-008)
-   On-call schedules — отдельный stack (ADR-008)

### Связи

ADR-001, ADR-006, ADR-008, FR-RUL-_, FR-NTF-_, FR-SIL-_, FR-SLO-_, QAS-AVAIL-01.

## 4.7. ADR-006: Visualization & UI

**ADR-006: Выбор UI / visualization layer**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-UI-001..005, FR-QRY-003 (correlation),
                      QAS-PERF-03, C-TECH-02
### Контекст

UI — главная точка контакта пользователей со СМА. Из FR-UI-001 следует требование unified UI для метрик/логов/трейсов/алертов с переключением контекста без потери выбора. UX напрямую влияет на MTTR (S3) и adoption платформы.

### Рассмотренные варианты

**Вариант 1: Grafana OSS**

Преимущества:

-   De-facto стандарт для observability UI
-   Native datasource plugins для Mimir, Loki, Tempo (ADR-001..003)
-   **Explore mode**  — лучшая в индустрии корреляция metrics↔logs↔traces (FR-QRY-003)
-   Dashboards-as-code (JSON, provisioning, Terraform provider) — FR-UI-002
-   Permalinks с полным контекстом (FR-UI-003) — встроенная функция
-   Огромная экосистема plugins, panels, dashboards
-   AGPLv3 — open source

Недостатки:

-   Mobile UX слабый (FR-UI-005 — challenge)
-   SLO/error budget UI требует доп. plugins (Pyrra, Sloth) или Grafana Enterprise
-   Multi-tenancy через organizations — workable, но не самая удобная
-   Под нагрузкой на сложных дашбордах может тормозить (требует tuning)

**Вариант 2: Grafana Enterprise**

Преимущества:

-   Дополнительные features: reporting, fine-grained RBAC, recorded queries, SLO panels
-   Enterprise support

Недостатки:

-   Платная — нарушает дух C-TECH-02 
-   Vendor lock-in на коммерческой основе
-   Большая часть critical features есть в OSS

**Вариант 3: Perses (CNCF)**

Преимущества:

-   Native dashboards-as-code first (single source of truth — YAML/JSON)
-   Современная архитектура
-   CNCF sandbox project

Недостатки:

-   Молодой продукт
-   Ограниченный набор datasource plugins
-   Нет explore-mode эквивалента
-   **Premature**  для нашего use case

**Вариант 4: Custom UI**

Недостатки:

-   Огромная стоимость разработки и сопровождения


### Решение

Выбран **Grafana OSS** как основной UI для СМА.

Дополнительно:

-   **Sloth + Pyrra dashboards**  для SLO views (FR-SLO-002)
-   **Grafana OnCall**  (см. ADR-008) для incident response UI
-   Mobile use case (FR-UI-005) — через Grafana mobile app (если стабилизуется) или через PagerDuty/OnCall mobile (приоритет)
-   Perses — мониторим как potential future replacement через 2-3 года

Обоснование:

-   **Single pane of glass**  для Mimir/Loki/Tempo через Explore — main UX driver (FR-UI-001, FR-QRY-003)
-   **Dashboards-as-code**  через JSON + Grafana Terraform provider удовлетворяет FR-UI-002
-   **Permalinks**  работают из коробки (FR-UI-003)
-   **AGPLv3**  соответствует C-TECH-02
-   Стоимость Enterprise не оправдана: критичные features (SLO, alerting) покрываются OSS + Sloth/Pyrra

### Последствия

Положительные:

-   Самый зрелый UX в индустрии для observability
-   Минимальная friction при использовании выбранного стека Mimir/Loki/Tempo
-   Богатая экосистема готовых dashboards (snowflake-style решения для типовых сервисов)

Отрицательные:

-   Mobile UX — слабая точка; критичный on-call UX (FR-UI-005) обеспечиваем через ADR-008 stack, не через Grafana
-   Grafana under high load требует tuning (caching, query timeouts) для QAS-PERF-03 / FR-UI-004
-   Зависимость от Grafana Labs roadmap (несмотря на OSS лицензию)

Нейтральные:

-   Авторизация через OIDC (FR-ADM-004) — supported, но требует настройки
-   HA развёртывание Grafana — нетривиально из-за state (alerts, annotations), используем shared DB (Postgres)

### Связи

ADR-001..005, ADR-008, FR-UI-*, FR-QRY-003.

## 4.8. ADR-007: Multi-tenancy Approach

**ADR-007: Стратегия multi-tenancy**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-ING-006, FR-ADM-001..003, QAS-SCAL-01, QAS-SEC-01, C-COST-01, C-ORG-02 (self-service)

### Контекст

Multi-tenancy — это **архитектурное решение поперёк всего стека**, не выбор технологии. Влияет на cost, security, blast radius и operational model. Тенанты в нашем случае — продуктовые команды и/или окружения (prod/staging/dev).

Три классических подхода: logical / physical / hybrid (cell-based).

### Рассмотренные варианты

**Вариант 1: Logical multi-tenancy (single shared cluster, tenant_id)**

Описание: один кластер Mimir/Loki/Tempo, тенанты разделены через `X-Scope-OrgID` header и tenant_id labels.

Преимущества:

-   Самый cost-efficient — shared infrastructure (C-COST-01)
-   Лучший resource utilization
-   Один deployment to operate — низкий operational overhead для маленькой SRE команды (C-ORG-01)
-   Mimir/Loki/Tempo поддерживают это нативно

Недостатки:

-   Noisy neighbor risk — митигируем через квоты (FR-ADM-002, QAS-SCAL-01)
-   Blast radius выше — bug в shared компоненте затрагивает всех
-   Security isolation через application-level checks, не infrastructure (мягче, чем physical)
-   Compliance: некоторые регуляторы требуют physical isolation (см. C-REG-02)

**Вариант 2: Physical multi-tenancy (cluster per tenant)**

Описание: отдельный кластер для каждого тенанта.

Преимущества:

-   Максимальная isolation (security, blast radius)
-   Простая compliance story
-   Tenant может иметь custom config

Недостатки:

-   **Не масштабируется**  на 50 команд — operational nightmare
-   Высокий cost (50x baseline overhead)
-   Дискриминирует мелкие команды (overprovision)
-   Нарушает C-COST-01

**Вариант 3: Hybrid / Cell-based**

Описание: тенанты сгруппированы в «cells» (например, 10 тенантов на cell); каждый cell — отдельный мини-кластер.

Преимущества:

-   Ограниченный blast radius (cell, не весь кластер)
-   Возможность размещать sensitive тенантов в dedicated cells
-   Лучше масштабируется, чем cluster-per-tenant
-   Data residency: можно размещать cells в разных регионах (C-REG-02)

Недостатки:

-   Сложность routing (какой tenant в каком cell?) — нужен tenant directory
-   Higher operational complexity, чем single cluster
-   Migration между cells нетривиальна

### Решение

Выбран **подход с эволюцией: logical multi-tenancy на старте → hybrid cell-based по мере роста**.

Конкретно:

1.  **Phase 1 (0-12 мес, до ~20 тенантов):**  один shared кластер Mimir/Loki/Tempo с logical isolation через tenant_id. Изоляция обеспечивается:
    -   Per-tenant квотами в Mimir/Loki/Tempo (FR-ADM-002)
    -   Rate limiting в OTel Gateway (ADR-004)
    -   RBAC в Grafana и API gateway (FR-ADM-003)
    -   Audit log (FR-ADM-005)
2.  **Phase 2 (12+ мес, либо при появлении sensitive tenants):**  разделение на 2-3 cells:
    -   **General cell**  — основной shared, для большинства команд
    -   **Sensitive cell**  — для compliance-критичных данных (audit logs, PII-heavy services)
    -   (Опционально)  **Region cell**  — для data residency (C-REG-02)

Обоснование:

-   **Phase 1**  даёт минимальную сложность для маленькой SRE команды (C-ORG-01) и максимальный cost-efficiency (C-COST-01)
-   Quotas + rate limits — достаточная защита от QAS-SCAL-01 для текущего масштаба
-   **Phase 2**  триггерится  **objective signals**: появление tenant с compliance requirements / blast radius incident / достижение ~20+ тенантов
-   Архитектура  **enables**  Phase 2, но не  **forces**  её преждевременно

### Последствия

Положительные:

-   Низкая сложность и cost в начале (operability fit для команды 5-7 человек)
-   Native поддержка в выбранных backends (Mimir, Loki, Tempo)
-   Path к cell-based при росте — не требует rewrite архитектуры

Отрицательные:

-   Phase 1 имеет higher blast radius — критичен self-monitoring (QAS-MON-01) для быстрого detection
-   Sensitive workloads должны ждать Phase 2 либо подключаться к external solutions (compromise на старте)
-   Quotas — critical, и tuning квот будет recurring operational задачей

Нейтральные:

-   Tenant onboarding (S1) включает квоту provisioning как обязательный шаг
-   Phase transition (Phase 1→2) требует data migration story — её нужно проработать до того, как Phase 2 станет необходимым (lead time)

### Связи

ADR-001..004, FR-ADM-*, QAS-SCAL-01, QAS-SEC-01, C-COST-01, C-REG-02.

## 4.9. ADR-008: On-Call & Incident Response

**ADR-008: Выбор on-call / incident response stack**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-OCM-001..003, FR-NTF-002..005, FR-UI-005,
                      C-ORG-03 (on-call rotation), C-TECH-02
### Контекст

On-call management и incident response — это надстройка над alerting (ADR-005). Нужны: расписания (FR-OCM-001), эскалация (FR-OCM-002), mobile-friendly ack/silence (FR-UI-005), voice/SMS notification (FR-NTF-002).

Это самостоятельная категория инструментов (PagerDuty, Opsgenie, VictorOps), а не часть metrics/logs stack.

### Рассмотренные варианты

**Вариант 1: PagerDuty**

Преимущества:

-   Industry standard, самый зрелый продукт
-   Превосходный mobile UX
-   Богатая интеграция с любыми системами
-   Incident management (timelines, postmortems) — лучшее в классе

Недостатки:

-   Дорогой ($21-41/user/month) — для команды 50+ накопительная стоимость значимая
-   **Proprietary SaaS**  — нарушает дух C-TECH-02
-   External dependency для critical path (alert routing)
-   Compliance может требовать on-premise

**Вариант 2: Opsgenie (Atlassian)**

Преимущества:

-   Дешевле PagerDuty
-   Интеграция с Atlassian stack (Jira, etc.)

Недостатки:

-   Proprietary SaaS (как PagerDuty)
-   Атлассианская стратегия неопределённая, Opsgenie merge'ится в JSM
-   Менее зрелый mobile UX, чем у PagerDuty

**Вариант 3: Grafana OnCall (OSS)**

Преимущества:

-   **Open source (AGPLv3)**  — соответствует C-TECH-02
-   Native интеграция с Grafana / Alertmanager (ADR-005, ADR-006)
-   Расписания, эскалация, voice/SMS (через Twilio integration)
-   Mobile app (iOS/Android)
-   Self-hosted — нет external dependency для critical path

Недостатки:

-   Younger product (~2021), менее зрелый, чем PagerDuty
-   Incident management (timelines, postmortems) слабее
-   Voice/SMS требует Twilio account (cost) — не самостоятельный
-   Активность разработки в 2024 неоднородна (нужно мониторить)

**Вариант 4: Custom build**

Недостатки: **Отклонён сразу** — это commodity, не conway-critical.

### Решение

Выбран **Grafana OnCall (self-hosted)** + **Twilio** для voice/SMS channel.

Обоснование:

-   **Open source + self-hosted**  — соответствует C-TECH-02 и убирает external SaaS dependency из critical path алертинга
-   **Native integration с Alertmanager и Grafana**  — нулевая friction с выбранным стеком (ADR-005, ADR-006)
-   **Cost-efficient**  для команды 50+: per-user pricing PagerDuty масштабируется накопительно, self-hosted — нет
-   **Mobile app**  покрывает FR-UI-005

PagerDuty — сильный alternative, и при появлении специфичных потребностей (advanced incident management, status pages) может быть рассмотрен. Но **Grafana OnCall достаточен для core use case** (FR-OCM-* + FR-NTF-*).

Постмортемы и incident timelines планируем покрыть **отдельным lightweight tooling** (Confluence templates / Jeli / Notion) — это не должно блокировать стек observability.

### Последствия

Положительные:

-   Унифицированный Grafana ecosystem (Mimir + Loki + Tempo + Alertmanager + OnCall) — один operational model
-   Нет vendor lock-in на critical path
-   Twilio как замещаемый компонент (можно сменить на любой SMS/voice provider)

Отрицательные:

-   Зависимость от health Grafana OnCall как community проекта — нужно мониторить трекшн
-   Incident management features beyond on-call (postmortems, status pages) — требуют дополнительных инструментов
-   Self-hosted Grafana OnCall требует ресурсов и operational ownership (но небольших)

Нейтральные:

-   Twilio costs — variable, зависят от volume voice/SMS (нужен бюджет)
-   В случае проблем с Grafana OnCall migration на PagerDuty — workable: API-based интеграции переключаемы

### Связи

ADR-005, ADR-006, FR-OCM-_, FR-NTF-_, FR-UI-005, C-ORG-03.

## 4.10. ADR-009: Storage Strategy & Retention

**ADR-009: Стратегия storage и retention**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-STO-001..004, QAS-RELI-01, QAS-AVAIL-02,
                      C-COST-01..03, C-TECH-04, C-REG-01..02

### Контекст

Storage — главный cost driver observability платформ (часто 60-80% TCO). Стратегия определяется: per-signal retention (FR-STO-001), tiering (hot/warm/cold) (FR-STO-002), replication (FR-STO-003), backup (FR-STO-004), durability (QAS-RELI-01), data residency (C-REG-02).

Это решение — **не single choice**, а композиция нескольких подрешений. Перечисляем их явно.

### Подрешение 1: Hot/Warm/Cold tiering

**Вариант A: Single-tier (всё в block storage)**

Преимущества: простота; быстрый query на всех окнах.

Недостатки: cost растёт линейно с retention; **отклонён** (нарушает C-COST-01/03).

**Вариант B: Hot (block) + Cold (S3) — 2 tier**

Преимущества: simple, native support в Mimir/Loki/Tempo (block storage SDKs).

Недостатки: query latency на cold tier выше, но это acceptable для редких use case (capacity planning, compliance).

**Вариант C: Hot + Warm + Cold — 3 tier**

Преимущества: finer-grained cost optimization.

Недостатки: больше moving parts; для нашего масштаба marginal benefit.

**Решение:** **Вариант B (2-tier: hot block + cold S3-compatible)**.

### Подрешение 2: Object storage provider

**Вариант A: AWS S3 (managed)**

Преимущества: industry standard, highest durability SLA (11 nines).

Недостатки: vendor lock-in на AWS.

**Вариант B: S3-compatible self-hosted (MinIO, Ceph)**

Преимущества: cloud-agnostic, on-prem capable.

Недостатки: операционная ответственность за durability.

**Вариант C: GCS / Azure Blob**

Преимущества: native в соответствующем облаке.

Недостатки: подобный AWS S3, но в другом cloud.

**Решение:** **S3 API как abstraction; конкретный provider — deployment-зависим** (AWS S3 для AWS, GCS для GCP, MinIO для on-prem). Mimir/Loki/Tempo все поддерживают S3 API — переключение тривиально (C-TECH-01 cloud-agnostic).

### Подрешение 3: Per-signal retention

Зафиксировано в FR-STO-001. Подтверждаем:


| Signal | Hot | Cold | Total |
| --- | --- | --- | --- |
| Metrics | 15 дней | 13 месяцев | 13 месяцев |
| Logs | 7 дней | 30 дней | 30 дней (audit — 2 года) |
| Traces | 3 дня | 7 дней | 7 дней |
| Audit log | — | 2 года | 2 года (immutable) |

### Подрешение 4: Replication & Durability

-   **Block storage (hot):**  RF=3 across 3 AZ → QAS-AVAIL-02
-   **Object storage (cold):**  native provider durability (S3 = 11 nines, MinIO с erasure coding)
-   **Audit log:**  synchronous replication, RF=3, immutability через S3 Object Lock или MinIO WORM

### Подрешение 5: Backup

-   **Configuration backup (rules, dashboards, SLO, tenants, RBAC):**  continuous backup в external object storage; RPO ≤ 5 минут (FR-DR-003)
-   **Data backup:**  не делаем отдельный (data уже в durable S3-tier); for catastrophic disaster (region loss) — cross-region replication object storage для audit log only

### Последствия

Положительные:

-   Cost в пределах C-COST-01..03 благодаря aggressive tiering
-   Native поддержка во всех выбранных backends
-   Cloud-agnostic через S3 API
-   Durability ≥ 99.99% для всех signals (QAS-RELI-01)

Отрицательные:

-   Query latency на cold tier (>15 дней metrics, >7 дней logs) выше — нужно явно коммуницировать пользователям
-   MinIO (если выбран) добавляет operational ownership (на AWS/GCP — managed)
-   Audit log immutability требует тщательной конфигурации Object Lock / WORM (FR-ADM-005, C-REG-01)

Нейтральные:

-   Data residency (C-REG-02) обеспечивается выбором региона object storage — конфигурация per-tenant в Phase 2 (см. ADR-007)
-   Configuration backup — отдельная operational задача (Velero / GitOps backup)

### Связи

ADR-001..003 (backends), ADR-007 (multi-tenancy → data residency), FR-STO-_, FR-DR-003, QAS-RELI-01, C-COST-_, C-REG-*.

## 4.11. ADR-010: GitOps & Configuration Management

**ADR-010: GitOps для конфигурации СМА**
─────────────────────────────────────
Статус:               Accepted
Дата:                 2024-XX-XX
Источники требований: FR-RUL-001 (rules-as-code), FR-UI-002 (dashboards-as-code),
                      FR-ADM-001 (tenant provisioning),
                      Maintainability tactics (3.6.5),
                      C-ORG-02 (self-service), C-ORG-04 (audit)


### Контекст

Конфигурация СМА состоит из множества артефактов: alerting rules, recording rules, SLO definitions, dashboards, tenant configs, RBAC, quotas, Alertmanager routes, OTel Collector configs. Если эти артефакты управляются вручную через UI — теряется auditability, reproducibility и self-service для продуктовых команд (S1, C-ORG-02).

GitOps означает: **Git — single source of truth** для конфигурации, изменения вносятся через PR, applied автоматически в кластер через reconciliation loop.

### Рассмотренные варианты

**Вариант 1: Argo CD**

Преимущества:

-   Самый зрелый GitOps tool в Kubernetes экосистеме
-   Богатый UI для visualisation sync state
-   Native поддержка Helm, Kustomize, plain YAML
-   ApplicationSet для multi-tenant pattern (одна Application per tenant)
-   Sync waves, hooks — fine control над rollout

Недостатки:

-   Self-contained UI — ещё один сервис для maintain
-   ApplicationSet pattern не тривиален для команд
-   Не специфичен для observability — нужны guidelines

**Вариант 2: Flux CD**

Преимущества:

-   Lightweight, более «native» для Kubernetes (CRDs only, без UI)
-   Лучше интегрируется с GitOps-as-code workflows
-   Native multi-tenancy через Kustomize overlays

Недостатки:

-   Слабее UI (хотя Weave GitOps OSS его частично закрывает)
-   Меньше adoption, чем Argo CD
-   Onboarding для команд без UI сложнее

**Вариант 3: Без GitOps — UI-based + manual provisioning**

Преимущества:

-   Низкий entry barrier
-   Не требует Git-fluency

Недостатки:

-   Нет auditability (нарушает C-ORG-04)
-   Нет reproducibility (test/staging/prod drift)
-   Self-service через UI — слабая security model
-   **Отклонён**: для платформы такого масштаба mandatory GitOps

**Вариант 4: Hybrid — GitOps для платформенных компонентов + UI для продуктовых команд**

Преимущества:

-   Низкий barrier для команд, не знакомых с GitOps
-   Платформенные изменения safe (через Git)

Недостатки:

-   Two paradigms — confusion и drift между ними
-   Невозможно дать команде «полный self-service с auditability»
-   **Отклонён**  как compromise hurting обоим стороны

### Решение

Выбран **Argo CD** для GitOps reconciliation + **monorepo подход для конфигурации**:
```text
sma-config/
  platform/                # управляется SRE командой
    mimir/
    loki/
    ...
  tenants/
    team-payments/
      ...
```
**Workflow:**

1.  Команда делает PR в свой  `tenants/team-X/`  каталог
2.  CI: валидация (promtool, sloth validate, dashboard linter)
3.  Code review от platform team для cross-tenant изменений; auto-approve для own-tenant rules/dashboards
4.  Merge → Argo CD reconciles в кластер за <5 мин
5.  Audit trail — Git history (C-ORG-04)

Обоснование:

-   **Argo CD**  — самый зрелый и UI-rich tool, что важно для onboarding команд (S1)
-   **Monorepo**  — единый audit trail, easier cross-tenant refactoring
-   **Path-based ownership**  через CODEOWNERS — self-service без compromise на security
-   **CI validation**  ловит ошибки до Argo CD sync — снижает blast radius bad config
-   Соответствует FR-RUL-001 (rules-as-code), FR-UI-002 (dashboards-as-code), C-ORG-02 (self-service), C-ORG-04 (audit)

### Последствия

Положительные:

-   Полный audit trail каждого изменения через Git
-   Reproducibility между средами (staging/prod parity)
-   Self-service для продуктовых команд без compromise на безопасность
-   Roll-back через  `git revert`  — простой и понятный
-   Platform team controls baseline через  `platform/`  каталог

Отрицательные:

-   Высокий entry barrier для команд, не знакомых с Git workflow — требует onboarding и templates (S1)
-   Initial setup CI validation pipelines — investment 2-4 недели
-   Dashboard JSON — verbose; разработка нового dashboard via JSON менее удобна, чем via UI; митигация: dashboard prototyping в Grafana UI, затем export в JSON через  `grafana-tools`
-   Schema migration (Mimir/Loki version upgrades с config changes) — координация через PR

Нейтральные:

-   Required tooling: Argo CD, GitHub/GitLab, CI runners (Argo Workflows / GitHub Actions)
-   Secrets management — отдельная сабтема (External Secrets Operator + Vault / AWS Secrets Manager)
-   Disaster recovery: восстановление конфигурации =  `git clone`  + Argo CD bootstrap (FR-DR-003)

### Связи

ADR-001..009 (всё, что имеет конфигурацию), FR-RUL-001, FR-UI-002, FR-ADM-001, Maintainability tactics (3.6.5), C-ORG-02, C-ORG-04.

## 4.12. Сводная таблица ADR

|ID|Решение |Альтернативы (отвергнутые)|Ключевой driver|
|------|-----------|--------|---------|
|ADR-001|Grafana Mimir|Thanos, VictoriaMetrics, Cortex|Multi-tenancy first-class + PromQL 100%|
|ADR-002|Grafana Loki|OpenSearch, ClickHouse, Elasticsearch|Cost + correlation с метриками|
|ADR-003|Grafana Tempo|Jaeger, ClickHouse|Cost + native correlation|
|ADR-004|OTel Collector (agent+gateway)|Vendor agents, Hybrid Fluent Bit|C-TECH-03 + unified pipeline|
|ADR-005|Mimir Ruler + Alertmanager + Sloth|Grafana Alerting unified, vmalert|Native Mimir + proven at scale|
|ADR-006|Grafana OSS|Grafana Enterprise, Perses | Maturity + AGPLv3|
|ADR-007|Logical → Hybrid cell-based|Physical per-tenant|Cost-fit на старте + path к isolation|
|ADR-008|Grafana OnCall + Twilio|PagerDuty, Opsgenie|Open source + no SaaS dependency|
|ADR-009|2-tier (block + S3), per-signal retention|Single-tier, 3-tier|Cost balance + native support|
|ADR-010|Argo CD + monorepo GitOps|Flux, UI-based, hybrid|Maturity + self-service UX|
||

## 4.13. Открытые вопросы и будущие ADR

Решения, отложенные до фазы Solution Architecture или последующих итераций:

| ID | Тема | Триггер для принятия |
| --- | --- | --- |
| ADR-011 | Sampling strategy (head vs tail) | После замеров trace volume в pilot |
| ADR-012 | Secrets management (Vault vs External Secrets vs Sealed Secrets) | До staging deployment |
| ADR-013 | Tenant directory / metadata service | При переходе к Phase 2 multi-tenancy |
| ADR-014 | Cost attribution / chargeback model | По запросу finance / C-COST-04 |
| ADR-015 | Synthetic monitoring (Grafana Synthetic vs Blackbox Exporter) | После coverage gap analysis |
| ADR-016 | Continuous profiling (Pyroscope / Parca) | При появлении explicit requirement |
| ADR-017 | Long-term audit log архив (S3 Glacier / tape) | По compliance review (C-REG-01) |

## 4.14. Влияние ADR на дальнейшие разделы

ADR определяют рамки для следующих разделов:

-   **Раздел 5 (Conceptual Architecture)**  — building blocks и их взаимодействия выводятся из ADR-001..006 (выбор движков)
-   **Раздел 6 (Component Architecture)**  — детальные компонентные диаграммы основываются на ADR-004 (collection), ADR-005 (alerting), ADR-007 (tenancy)
-   **Раздел 7 (Cross-cutting Concerns)**  — security, observability-of-observability, DR опираются на ADR-007..010
-   **Раздел 8 (Deployment Architecture)**  — K8s топология, sizing, scaling выводятся из ADR-001..003, ADR-009
-   **Раздел 9 (Operational Model)**  — runbook'и, on-call, incident response основываются на ADR-005, ADR-008, ADR-010
-   **Раздел 10 (Roadmap)**  — Phase 1→2 transitions (multi-tenancy, sampling) синхронизированы с ADR-007 и open ADRs

## 4.15. Резюме раздела

Зафиксированы **10 ключевых архитектурных решений**, которые определяют технологический стек СМА:

**Grafana-centric observability stack:**

-   Mimir (metrics) + Loki (logs) + Tempo (traces) + Grafana (UI) + OnCall + Alertmanager
-   OpenTelemetry Collector как unified collection layer
-   Sloth для SLO-as-code
-   Argo CD для GitOps

**Сквозные принципы:**

-   Multi-tenancy first-class на всех уровнях (logical на старте, эволюция к cell-based)
-   Object storage (S3 API) как primary cold tier — radical cost reduction
-   Open source preference (AGPLv3/Apache 2.0 — все компоненты) — C-TECH-02
-   Cloud-agnostic через стандартные API (S3, OTLP, Kubernetes) — C-TECH-01
-   Декларативная конфигурация через Git как single source of truth — C-ORG-02, C-ORG-04

**Сознательные компромиссы:**

-   Operational complexity > Cost-effectiveness (Mimir > VictoriaMetrics)
-   Cost-effectiveness > Search expressiveness (Loki > OpenSearch)
-   Maturity > Innovation (Grafana OSS > Perses)
-   Self-hosted > Managed (Grafana OnCall > PagerDuty)

7 будущих ADR зафиксированы как known-unknowns с триггерами принятия.