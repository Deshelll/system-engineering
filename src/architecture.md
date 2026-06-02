# 4. Architecture Decisions (ADRs)
## 4.1. Назначение раздела
Раздел описывает архитектуру системы мониторинга и алертинга (СМА) на
уровне компонентов, обязанностей, потоков взаимодействия, размещения в
Kubernetes, отказоустойчивости и архитектурных решений. Архитектура
выводится из CONOPS, функциональных требований, нефункциональных
требований и ограничений проекта. Основная цель раздела — показать,
какие подсистемы реализуют требования, как они взаимодействуют, какие
данные хранят и каким образом система сохраняет критичные функции при
отказах.

В отличие от перечня технологий, архитектура фиксирует устойчивые
проектные решения: границы компонентов, распределение ответственности,
состояние системы, механизмы масштабирования, деградации и
восстановления. Отдельные решения оформлены как ADR, поскольку их
изменение после внедрения потребует значительных затрат на миграцию,
тестирование и эксплуатационную перестройку.

**Формат:** упрощённый MADR.

**Общий контекст для всех ADR:**
- Целевая среда: Kubernetes (cloud-agnostic, предпочтение AWS/GCP)

- Масштаб: 10-50 продуктовых команд, 100-500 сервисов

- Подход: build (self-hosted), strong preference for open source

- Заказчик: greenfield с возможным легаси-Prometheus

## 4.2. Архитектурные драйверы и ограничения
Архитектурные решения принимаются с учётом следующих драйверов. Они
используются как критерии оценки альтернатив в ADR и как основание для
декомпозиции системы.

| ID    | Драйвер                                                 | Архитектурное следствие                                                                                                  |
|-------|---------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| AD-01 | Поддержка около 500 сервисов и дальнейший рост нагрузки | Компоненты проектируются горизонтально масштабируемыми; stateful-части используют репликацию и независимые storage-слои. |
| AD-02 | Снижение MTTD и MTTR                                    | Alerting path, корреляция metrics → logs → traces и быстрые UI-запросы имеют повышенный приоритет.                       |
| AD-03 | Multi-tenancy и изоляция команд                         | Все данные несут tenant_id; доступ контролируется на уровне API, UI, storage и query.                                    |
| AD-04 | Предпочтение open-source и cloud-native подхода         | Выбор делается в пользу компонентов, которые можно развернуть self-hosted в Kubernetes.                                  |
| AD-05 | Контроль стоимости observability                        | Используются tiered storage, квоты, sampling и ограничение high-cardinality данных.                                      |
| AD-06 | Критичность alerting path                               | При деградации UI/query система сохраняет ingestion метрик, evaluation правил и доставку P1/P2-алертов.                  |
| AD-07 | Воспроизводимость конфигурации                          | Rules, dashboards, SLO, routing policies и tenant-конфигурация хранятся как код и применяются через GitOps.              |
| AD-08 | Восстановимость после сбоя                              | Конфигурация резервируется с RPO ≤ 5 минут; базовая функциональность восстанавливается по runbook с RTO ≤ 15 минут.      |

## 4.3. Высокоуровневая архитектура
СМА строится как набор слабо связанных подсистем, развёрнутых в
Kubernetes. Входной слой принимает телеметрию от продуктовой платформы,
нормализует и маршрутизирует её по tenant-ам. Storage-слой хранит
метрики, логи и трейсы в специализированных backends. Rules Engine и
Alertmanager формируют и обрабатывают алерты. Notification и On-call
подсистемы отвечают за доставку уведомлений, подтверждение и эскалацию.
UI/API слой предоставляет пользователям единое окно расследования,
администрирования и просмотра состояния системы.

Логическая схема потоков:

Продуктовые сервисы → OTel Agent → OTel Gateway → Tenant Router → Mimir
/ Loki / Tempo

Mimir Ruler → Alertmanager → Notification Dispatcher → Slack / Push /
SMS / Voice / Webhook

Grafana UI / API Gateway → Query Services → Mimir / Loki / Tempo

Git repository → CI validation → Argo CD → Kubernetes resources and
configuration

Meta-monitoring → independent notification channel → Platform on-call

Данная схема разделяет пять контуров: сбор телеметрии, хранение и
запросы, алертинг, пользовательский доступ и управление конфигурацией.
Такое разделение уменьшает связность компонентов и позволяет
деградировать отдельные функции без полной остановки СМА.

## 4.4. Декомпозиция компонентов и обязанности
Ключевые компоненты СМА и их обязанности приведены в таблице. Для
каждого компонента указаны входы, выходы, наличие состояния, способ
масштабирования и связанные требования. Такая декомпозиция фиксирует
границы ответственности и предотвращает дублирование функций между
подсистемами.

| Компонент               | Ответственность                                                      | Вход                                               | Выход                                   | Состояние / масштабирование                                        | Трассировка                    |
|-------------------------|----------------------------------------------------------------------|----------------------------------------------------|-----------------------------------------|--------------------------------------------------------------------|--------------------------------|
| OTel Agent              | Сбор локальной телеметрии с pod/node; первичная буферизация.         | Метрики, логи, трейсы приложений и инфраструктуры. | OTLP batches в OTel Gateway.            | DaemonSet; локальный buffer; масштабируется числом узлов.          | FR-ING-001–005                 |
| OTel Gateway            | Tenant routing, rate limiting, redaction, batching, backpressure.    | OTLP/gRPC, OTLP/HTTP.                              | Потоки в Mimir, Loki, Tempo.            | Stateless + очередь; HPA по CPU, memory, queue length.             | FR-ING-004–006, NFR-PERF-001   |
| Tenant Router           | Проверка tenant_id, применение квот, выбор backend/cell.             | Обогащённая телеметрия.                            | Маршрутизированные tenant-aware потоки. | Stateless; хранит правила в конфигурации GitOps.                   | FR-ADM-001–002, FR-ING-006     |
| Metrics Store (Mimir)   | Хранение временных рядов, PromQL-запросы, long retention, ruler.     | Remote Write / OTLP metrics.                       | PromQL results, alert evaluations.      | Stateful; RF=3; horizontal scaling.                                | FR-STO-001–003, FR-QRY-001     |
| Log Store (Loki)        | Хранение и поиск логов по labels и тексту.                           | Log streams.                                       | LogQL/search results.                   | Stateful; RF=3; object storage tier.                               | FR-STO-001–004, FR-QRY-002     |
| Trace Store (Tempo)     | Хранение traces, поиск по trace_id, связь с логами.                  | OTLP spans.                                        | Trace waterfall, span attributes.       | Stateful; object storage; sampling на ingestion.                   | FR-ING-003, FR-QRY-003         |
| Correlation Service     | Формирование контекста alert → metrics/logs/traces/deploy events.    | Alert context, service metadata, time range.       | Ссылки и фильтры для UI/API.            | Stateless; кэш metadata с TTL.                                     | FR-QRY-003, FR-UI-001          |
| Rule Engine / Ruler     | Вычисление alert rules, recording rules и burn-rate rules.           | Metrics queries, rules as code.                    | Alert instances.                        | HA deployment; конфигурация из Git.                                | FR-RUL-001–003, FR-SLO-001–002 |
| Alertmanager            | Группировка, дедупликация, inhibition, silence, lifecycle алертов.   | Alert instances.                                   | Notification events.                    | HA cluster; replicated alert state.                                | FR-RUL-004–005, FR-SIL-001–003 |
| Notification Dispatcher | Retry, fallback, DLQ, отправка во внешние каналы.                    | Notification events.                               | Slack/email/push/SMS/voice/webhook.     | Stateless workers + durable queue/DLQ; HPA.                        | FR-NTF-001–005                 |
| On-call Service         | Расписания, primary/secondary responder, swap, override, escalation. | Routing policies, schedule changes, ack.           | Responder selection, escalation steps.  | PostgreSQL-backed state; HA application replicas.                  | FR-OCM-001–003                 |
| API Gateway             | Единая точка API, authN/authZ, rate limiting, OpenAPI contracts.     | HTTPS/gRPC external requests.                      | Запросы к внутренним сервисам.          | Stateless; HPA; policy config из Git.                              | FR-API-001–003, FR-ADM-003–004 |
| Grafana UI              | Единый интерфейс metrics/logs/traces/alerts/SLO; permalinks.         | Пользовательские запросы.                          | Dashboards, explore, alert context.     | Stateless web replicas + shared DB.                                | FR-UI-001–005                  |
| Audit Service           | Фиксация критичных действий и изменений конфигурации.                | Events from UI/API/Alertmanager/OnCall.            | Immutable audit log.                    | Append-only storage; retention ≥ 2 года.                           | FR-ADM-005                     |
| Backup / DR Service     | Backup конфигурации, restore drill, восстановление schedules/RBAC.   | Git state, DB snapshots, object storage.           | Restore artifacts, DR reports.          | Scheduled jobs; external object storage.                           | FR-DR-003–004                  |
| Meta-monitoring Stack   | Независимый контроль состояния СМА и synthetic checks.               | Health metrics, synthetic probes.                  | Critical alerts в независимый канал.    | Отдельный namespace/cell; минимальная зависимость от основной СМА. | FR-DR-001, QAS-MON-01          |

## 4.5. Runtime-представления ключевых сценариев
### 4.5.1. Приём телеметрии и маршрутизация tenant-а
1\. Продуктовый сервис отправляет метрики, логи и трейсы через
OpenTelemetry SDK, Prometheus Remote Write или syslog-совместимый
источник.

2\. OTel Agent на узле собирает локальные сигналы и передаёт их в OTel
Gateway.

3\. OTel Gateway проверяет наличие tenant_id и service_id, добавляет
технические метаданные окружения и применяет rate limiting.

4\. Tenant Router определяет, к какому tenant/cell относится поток, и
направляет его в соответствующий backend.

5\. Metrics Store, Log Store и Trace Store подтверждают приём только
после записи в надёжный слой или после постановки в durable очередь.

6\. При превышении квоты включается backpressure; отклонённые данные
фиксируются в метриках и audit/operational logs без молчаливой потери.

### 4.5.2. Срабатывание алерта и эскалация
1\. Rule Engine периодически вычисляет правила по метрикам и SLO
burn-rate rules.

2\. При выполнении условия создаётся alert instance с tenant_id,
service_id, severity, environment и ссылкой на runbook.

3\. Alertmanager группирует алерты, применяет дедупликацию, inhibition и
проверяет активные silence.

4\. Notification Dispatcher получает notification event и выбирает канал
доставки согласно routing policy.

5\. On-call Service определяет primary responder и, при отсутствии ack,
инициирует переход к secondary responder и team lead.

6\. Ack через UI, mobile-friendly интерфейс или внешний канал обновляет
alert lifecycle и останавливает дальнейшую эскалацию.

7\. Для P1-инцидентов создаётся внешний incident ticket через webhook;
все действия фиксируются в audit log.

### 4.5.3. Расследование инцидента metrics → logs → traces
1\. On-call инженер открывает alert detail page из уведомления.

2\. UI получает alert context и формирует запросы к Mimir, Loki и Tempo
с сохранением tenant_id, service_id, environment и временного окна.

3\. Correlation Service добавляет deploy markers и metadata владельца
сервиса из Service Catalog cache.

4\. Пользователь переходит от графика метрики к логам и trace без
ручного повторного ввода фильтров.

5\. Permalink сохраняет состояние расследования и может быть передан
другому пользователю с теми же правами.

### 4.5.4. GitOps-изменение конфигурации
1\. Команда изменяет alert rules, SLO, dashboards или tenant quotas в
Git-репозитории.

2\. CI выполняет статическую проверку: schema validation, promtool/sloth
validation, lint dashboard definitions и проверку ownership.

3\. После merge Argo CD синхронизирует конфигурацию с Kubernetes и
внутренними API СМА.

4\. Результат применения фиксируется в audit log; при ошибке изменения
не попадают в production или откатываются через git revert.

## 4.6. Управление состоянием и владение данными
Архитектура явно разделяет stateless-компоненты, stateful-компоненты и
внешние источники истины. Это необходимо для корректного восстановления,
масштабирования и анализа отказов.

| Данные / состояние                                      | Источник истины             | Владелец                             | Backup / retention                                 | Критичность |
|---------------------------------------------------------|-----------------------------|--------------------------------------|----------------------------------------------------|-------------|
| Rules, dashboards, SLO, routing policies, tenant quotas | Git monorepo                | Platform team + CODEOWNERS tenant-ов | Git history; восстановление через Argo CD          | Высокая     |
| Метрики                                                 | Mimir + object storage tier | СМА                                  | Retention 13 месяцев; RF=3 для hot tier            | Высокая     |
| Логи                                                    | Loki + object storage tier  | СМА                                  | Retention по типу логов; audit отдельно ≥ 2 года   | Высокая     |
| Трейсы                                                  | Tempo + object storage tier | СМА                                  | Короткий retention; sampling policy                | Средняя     |
| Alert lifecycle                                         | Alertmanager / On-call DB   | СМА                                  | HA state + DB backup                               | Критическая |
| On-call schedules, overrides, swaps                     | On-call DB + Git export     | Team Lead / Platform Admin           | Backup ≤ 5 минут                                   | Критическая |
| Audit log                                               | Append-only audit storage   | Compliance / Security                | Immutable retention ≥ 2 года                       | Критическая |
| Service ownership metadata                              | Service Catalog / CMDB      | Внешняя команда                      | TTL cache в СМА                                    | Высокая     |
| Identity and groups                                     | IdP / SSO                   | IAM-команда                          | Cached sessions; break-glass доступ                | Высокая     |
| DLQ уведомлений                                         | Notification queue storage  | СМА                                  | Хранение до ручного разбора или повторной отправки | Высокая     |

## 4.7. Deployment architecture
СМА разворачивается в Kubernetes в нескольких namespace-ах. Разделение
namespace-ов соответствует функциональным границам компонентов и
упрощает применение NetworkPolicy, квот ресурсов и операционного
мониторинга.

| Namespace           | Компоненты                                                            | Назначение                                          |
|---------------------|-----------------------------------------------------------------------|-----------------------------------------------------|
| sma-collection      | OTel Agent, OTel Gateway, Tenant Router                               | Приём, нормализация и маршрутизация телеметрии.     |
| sma-metrics         | Mimir distributor, ingester, querier, ruler, compactor, store-gateway | Хранение метрик, PromQL-запросы, evaluation правил. |
| sma-logs            | Loki distributor, ingester, querier, compactor                        | Хранение и поиск логов.                             |
| sma-traces          | Tempo distributor, ingester, querier, compactor                       | Хранение и просмотр трейсов.                        |
| sma-alerting        | Alertmanager, Notification Dispatcher, Grafana OnCall                 | Обработка алертов, доставка уведомлений, эскалация. |
| sma-visualization   | Grafana OSS, API Gateway                                              | Пользовательский UI и внешний API.                  |
| sma-gitops          | Argo CD, repo server, application controller                          | Синхронизация конфигурации из Git.                  |
| sma-backup          | Backup jobs, restore jobs, Velero-compatible jobs                     | Резервное копирование и восстановление.             |
| sma-meta-monitoring | Minimal metrics stack, synthetic probes, independent notifier         | Независимый мониторинг самой СМА.                   |

Базовые правила размещения:

- Критичные stateless-компоненты имеют не менее 3 реплик и
  распределяются по availability zones через pod anti-affinity.

- Stateful-компоненты используют replication factor 3 и размещаются
  zone-aware, чтобы потеря одной зоны не приводила к потере
  подтверждённых данных.

- UI и query layer допускают деградацию; ingestion метрик, evaluation
  правил и доставка P1/P2-алертов имеют повышенный приоритет.

- HPA включается для OTel Gateway, distributors, query components,
  Notification Dispatcher и API Gateway.

- Object storage используется как общий холодный слой и как место
  хранения backup-артефактов.

- Для on-prem deployment используется S3-compatible storage, например
  MinIO или Ceph; для cloud deployment — managed object storage
  провайдера.

## 4.8. Отказоустойчивость и graceful degradation
Архитектура проектируется исходя из принципа приоритета критичных
функций. При частичном отказе система не должна деградировать
симметрично: функции обнаружения и доставки критичных алертов
сохраняются дольше, чем визуализация и ad-hoc расследование.

| Отказ                         | Ожидаемое поведение                                   | Компенсация / деградация                                                                    | Проверка                 |
|-------------------------------|-------------------------------------------------------|---------------------------------------------------------------------------------------------|--------------------------|
| Отказ Grafana UI              | Alerting и ingestion продолжают работать.             | Пользователи используют API, direct links или mobile/on-call UI.                            | TC graceful degradation  |
| Отказ query layer             | Приём телеметрии и evaluation правил сохраняются.     | UI показывает degraded state; расследование ограничено.                                     | Chaos test               |
| Отказ log storage             | Метрики и alerting продолжают работать.               | Логи буферизуются или принимаются с ограничением; low-priority logs могут rate-limit-иться. | Storage failure test     |
| Отказ trace storage           | Метрики, логи и alerting сохраняются.                 | Trace viewer показывает partial/unavailable status.                                         | Chaos test               |
| Отказ основного chat-канала   | Notification Dispatcher переходит к fallback-каналам. | Push → SMS → voice → team lead; событие в DLQ при полном отказе.                            | TC-10                    |
| Недоступность Service Catalog | Используется cached ownership metadata.               | TTL cache; fallback на platform on-call при истечении TTL.                                  | Integration failure test |
| Недоступность IdP             | Существующие sessions продолжают действовать.         | Break-glass admin для emergency; новые входы ограничены.                                    | Auth failure drill       |
| Отказ одной availability zone | СМА сохраняет базовую функциональность.               | Traffic reroute на оставшиеся зоны; rebalance storage.                                      | AZ failure drill         |
| Отказ primary deployment      | Failover на warm standby.                             | Restore из Git и backup; catch-up notification по окну недоступности.                       | DR drill                 |

## 4.9. Безопасность и multi-tenancy
Безопасность реализуется не отдельным компонентом, а сквозным набором
архитектурных механизмов. Основной принцип: tenant_id является
обязательным контекстом для ingestion, storage, query, alerting, UI и
audit.

| Механизм                        | Где применяется                               | Что предотвращает                                              |
|---------------------------------|-----------------------------------------------|----------------------------------------------------------------|
| Tenant-aware API Gateway        | Все внешние API-запросы                       | Подмену tenant_id и доступ к чужим ресурсам.                   |
| RBAC на уровне tenant и ресурса | UI, API, admin operations                     | Несанкционированное изменение rules, silence, schedules, RBAC. |
| Per-tenant quotas               | OTel Gateway, Mimir, Loki, Tempo, query layer | Noisy neighbor, cardinality explosion, cost overrun.           |
| NetworkPolicy deny-by-default   | Kubernetes namespaces                         | Ненужные межкомпонентные соединения.                           |
| mTLS / TLS in-transit           | Внешние и внутренние соединения               | Перехват и изменение данных в канале передачи.                 |
| Redaction sensitive data        | Ingestion pipeline                            | Запись PII/secrets в persisted storage.                        |
| Immutable audit log             | Все критичные действия                        | Сокрытие изменений и невозможность расследования.              |
| Break-glass доступ              | Emergency admin operations                    | Полную блокировку администрирования при сбое IdP.              |

## 4.10. Архитектурная трассировка требований к компонентам
| Группа требований | Основные компоненты                                             | Архитектурная реализация                                                               |
|-------------------|-----------------------------------------------------------------|----------------------------------------------------------------------------------------|
| FR-ING-\*         | OTel Agent, OTel Gateway, Tenant Router                         | Приём OTLP/Prometheus/syslog, enrichment, backpressure, rate limiting, tenant routing. |
| FR-STO-\*         | Mimir, Loki, Tempo, Object Storage Adapter                      | Раздельное хранение сигналов, retention, tiered storage, replication.                  |
| FR-QRY-\*         | Grafana UI, API Gateway, Query components, Correlation Service  | PromQL/LogQL/trace lookup, сохранение контекста alert → investigation.                 |
| FR-RUL-\*         | Mimir Ruler, Alertmanager, Sloth/Pyrra-compatible SLO generator | Rules-as-code, burn-rate rules, grouping, deduplication, inhibition.                   |
| FR-NTF-\*         | Notification Dispatcher, Channel Adapters, On-call Service      | Routing policies, retry, fallback, DLQ, escalation chains.                             |
| FR-OCM-\*         | On-call Service, Grafana OnCall DB, Audit Service               | Schedules, swap, override, primary/secondary responder, audit.                         |
| FR-SLO-\*         | SLO Engine, Mimir, Grafana dashboards                           | SLI calculation, error budget, burn rate, SLO dashboard.                               |
| FR-SIL-\*         | Alertmanager, UI/API, Audit Service                             | Silence lifecycle, auto-expire, 4-eyes approval, audit.                                |
| FR-API-\*         | API Gateway, internal service APIs, OpenAPI specification       | Документированный внешний API и outbound webhooks.                                     |
| FR-UI-\*          | Grafana OSS, Correlation Service, On-call UI                    | Unified UI, permalinks, mobile-friendly ack.                                           |
| FR-ADM-\*         | Tenant Service, RBAC Engine, IdP Adapter, Audit Service         | Tenant lifecycle, квоты, RBAC, SSO/OIDC, immutable audit.                              |
| FR-DR-\*          | Backup Service, Meta-monitoring Stack, Argo CD, Object Storage  | Backup, restore, independent monitoring, DR drill.                                     |

## 4.11. Architecture Decision Records
В этом подразделе зафиксированы ключевые архитектурные решения. Для
сопоставимости альтернатив используется единая шкала: 0 — критерий не
выполняется; 1 — выполняется частично; 2 — выполняется полностью.
Итоговая оценка рассчитывается как сумма произведений веса критерия на
оценку альтернативы. Выбранная альтернатива не обязана иметь абсолютный
максимум по каждому критерию, но должна давать наилучший баланс для
ограничений проекта.

### ADR-001. Выбор backend для метрик
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. Метрики являются основой alerting, SLO и capacity planning.
Backend должен поддерживать PromQL, multi-tenancy, горизонтальное
масштабирование и long retention при целевой нагрузке до 10x от
стартового уровня.

| Критерий                              | Вес | Mimir | Thanos | VictoriaMetrics |
|---------------------------------------|-----|-------|--------|-----------------|
| PromQL-совместимость                  | 5   | 2     | 2      | 1               |
| First-class multi-tenancy             | 5   | 2     | 1      | 1               |
| Горизонтальное масштабирование        | 4   | 2     | 2      | 2               |
| Object storage / long retention       | 4   | 2     | 2      | 1               |
| Операционная сложность для старта     | 3   | 1     | 1      | 2               |
| Соответствие выбранному Grafana-stack | 3   | 2     | 1      | 1               |
| Итоговая взвешенная оценка            |     | 45    | 36     | 31              |

Решение. Выбран Grafana Mimir, поскольку он лучше закрывает два наиболее
весомых критерия: PromQL-совместимость и first-class multi-tenancy. Для
снижения операционной сложности используется поэтапный запуск:
упрощённая топология на старте и переход к полной
microservices-топологии при росте нагрузки.

Положительные последствия:

- Единая модель tenant_id для метрик, алертов и query.

- Поддержка long retention через object storage.

- Упрощение интеграции с Grafana и Mimir Ruler.

Отрицательные последствия и меры контроля:

- Операционная сложность выше, чем у более компактных решений;
  компенсируется Helm-чартами, runbook-ами и staged rollout.

- Cardinality требует строгих квот; контроль реализуется через
  FR-ADM-002 и QAS-SCAL-01.

### ADR-002. Выбор backend для логов
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. Логи имеют наибольший объём среди observability-сигналов.
Основной сценарий — поиск логов известного сервиса за известное
временное окно с дальнейшей корреляцией с метриками и трейсами.

| Критерий                               | Вес | Loki | OpenSearch | ClickHouse |
|----------------------------------------|-----|------|------------|------------|
| Стоимость хранения при больших объёмах | 5   | 2    | 0          | 2          |
| Корреляция с metrics/traces в Grafana  | 4   | 2    | 1          | 1          |
| Full-text search без labels            | 3   | 1    | 2          | 1          |
| First-class multi-tenancy              | 4   | 2    | 1          | 1          |
| Операционная сложность                 | 3   | 1    | 1          | 1          |
| Совместимость с выбранным стеком       | 3   | 2    | 1          | 1          |
| Итоговая взвешенная оценка             |     | 43   | 24         | 29         |

Решение. Выбран Grafana Loki как основной backend для логов. OpenSearch
не исключается полностью, но рассматривается как дополнительный
специализированный контур для задач, где требуется произвольный
полнотекстовый поиск по audit/security-логам.

Положительные последствия:

- Снижение стоимости хранения логов за счёт label-based indexing и
  object storage.

- Единый Grafana UX для переходов metrics → logs → traces.

- Единая multi-tenant модель с Mimir и Tempo.

Отрицательные последствия и меры контроля:

- Поиск без корректных labels может быть медленным; компенсируется
  стандартом labeling и onboarding checklist.

- Для security/audit use case может потребоваться отдельный
  OpenSearch-контур; решение оформляется отдельным ADR при появлении
  требования.

### ADR-003. Выбор backend для трейсов
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. Трейсы используются в сценарии расследования инцидента, прежде
всего для поиска trace по trace_id и анализа waterfall. Из-за большого
объёма spans требуется короткий retention и sampling strategy.

| Критерий                          | Вес | Tempo | Jaeger | ClickHouse |
|-----------------------------------|-----|-------|--------|------------|
| Быстрый поиск по trace_id         | 5   | 2     | 2      | 1          |
| Стоимость хранения                | 4   | 2     | 1      | 2          |
| Интеграция с Grafana/Mimir/Loki   | 4   | 2     | 1      | 1          |
| Attribute search                  | 3   | 1     | 2      | 2          |
| Multi-tenancy                     | 3   | 2     | 1      | 1          |
| Операционная унификация со стеком | 3   | 2     | 1      | 1          |
| Итоговая взвешенная оценка        |     | 45    | 34     | 32         |

Решение. Выбран Grafana Tempo. Основной критерий — эффективный trace_id
lookup и тесная интеграция с Grafana Explore. Сложный поиск по
attributes не является стартовым доминирующим use case и может быть
усилен позже отдельным индексным слоем.

Положительные последствия:

- Дешёвое хранение высокообъёмных traces.

- Нативная корреляция с логами и метриками.

- Единая operational model для observability backends.

Отрицательные последствия и меры контроля:

- Attribute-based search ограничен; компенсируется сохранением trace_id
  в логах и корректной propagation.

- Требуется явная sampling policy в OTel Gateway.

### ADR-004. Выбор telemetry collection layer
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. Collection layer является входной точкой СМА и влияет на
onboarding, backpressure, redaction и tenant routing. Требуется единый
стандарт для метрик, логов и трейсов.

| Критерий                                       | Вес | OTel Agent+Gateway | Vendor agents | Hybrid Fluent Bit + OTel |
|------------------------------------------------|-----|--------------------|---------------|--------------------------|
| Единый стандарт для всех сигналов              | 5   | 2                  | 0             | 1                        |
| Vendor-neutral                                 | 4   | 2                  | 0             | 2                        |
| Централизованные политики redaction/rate limit | 4   | 2                  | 1             | 1                        |
| Эффективность log collection                   | 3   | 1                  | 2             | 2                        |
| Операционная простота                          | 3   | 2                  | 1             | 1                        |
| Backpressure и batching                        | 3   | 2                  | 1             | 1                        |
| Итоговая взвешенная оценка                     |     | 47                 | 16            | 31                       |

Решение. Выбрана топология OpenTelemetry Collector Agent + OpenTelemetry
Collector Gateway. Agent разворачивается как DaemonSet, Gateway — как
масштабируемый Deployment с tenant routing, redaction, batching и
backpressure.

Положительные последствия:

- Единая точка политики для всех сигналов.

- Снижение зависимости от backend-специфичных агентов.

- Возможность централизованно применять redaction и rate limiting.

Отрицательные последствия и меры контроля:

- Resource footprint выше, чем у специализированных агентов;
  контролируется нагрузочным тестированием и HPA.

- При экстремальном объёме логов может потребоваться добавление Fluent
  Bit как исключение, что оформляется отдельным ADR.

### ADR-005. Выбор alerting stack
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. Alerting path является критичной функцией системы. Требуется
декларативное описание правил, поддержка grouping, deduplication,
silence, inhibition и burn-rate SLO alerting.

| Критерий                          | Вес | Mimir Ruler + Alertmanager + Sloth | Grafana Alerting | Custom evaluator |
|-----------------------------------|-----|------------------------------------|------------------|------------------|
| PromQL rules at scale             | 5   | 2                                  | 1                | 1                |
| Grouping/dedup/silence/inhibition | 5   | 2                                  | 1                | 1                |
| Rules-as-code                     | 4   | 2                                  | 1                | 2                |
| SLO burn-rate generation          | 4   | 2                                  | 1                | 1                |
| Операционная зрелость             | 3   | 2                                  | 1                | 0                |
| Стоимость разработки              | 3   | 2                                  | 2                | 0                |
| Итоговая взвешенная оценка        |     | 48                                 | 27               | 23               |

Решение. Выбран Mimir Ruler + Prometheus Alertmanager + Sloth-compatible
генерация SLO-правил. Grafana Alerting может использоваться точечно для
multi-datasource алертов, но не как основной alerting path.

Положительные последствия:

- Стандартная модель alert lifecycle и routing.

- Декларативная конфигурация правил через GitOps.

- Поддержка SLO burn-rate без custom evaluator.

Отрицательные последствия и меры контроля:

- Несколько компонентов вместо одного UI; компенсируется документацией
  границ ответственности.

- Voice/SMS и on-call schedules вынесены в отдельную подсистему,
  описанную в ADR-008.

### ADR-006. Выбор visualization layer
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. UI должен поддерживать единый сценарий расследования: переход
от алерта к метрикам, логам, трейсам и SLO без потери
tenant/service/time context.

| Критерий                   | Вес | Grafana OSS | Grafana Enterprise | Custom UI |
|----------------------------|-----|-------------|--------------------|-----------|
| Поддержка Mimir/Loki/Tempo | 5   | 2           | 2                  | 0         |
| Unified investigation UX   | 5   | 2           | 2                  | 1         |
| Dashboards-as-code         | 4   | 2           | 2                  | 1         |
| Лицензирование / cost fit  | 4   | 2           | 0                  | 1         |
| Срок внедрения             | 3   | 2           | 2                  | 0         |
| Полный контроль UX         | 2   | 1           | 1                  | 2         |
| Итоговая взвешенная оценка |     | 45          | 37                 | 13        |

Решение. Выбран Grafana OSS как основной UI. Для on-call mobile use case
применяется не основной Grafana UI, а On-call подсистема и
mobile-friendly ack path.

Положительные последствия:

- Минимальная стоимость внедрения и высокая совместимость с выбранными
  backends.

- Поддержка permalinks и dashboards-as-code.

- Снижение риска разработки собственного UI.

Отрицательные последствия и меры контроля:

- Mobile UX ограничен; критичный ack реализуется через On-call Service и
  внешние каналы.

- Требуется настройка RBAC, organizations/folders и datasource
  permissions.

### ADR-007. Стратегия multi-tenancy
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. Тенанты соответствуют продуктовым командам и/или окружениям.
Требуется баланс между стоимостью, изоляцией, blast radius и сложностью
сопровождения.

| Критерий                     | Вес | Logical → cell-based | Physical per tenant | Single shared без cell path |
|------------------------------|-----|----------------------|---------------------|-----------------------------|
| Стоимость на старте          | 5   | 2                    | 0                   | 2                           |
| Security isolation           | 5   | 1                    | 2                   | 1                           |
| Операционная сложность       | 4   | 2                    | 0                   | 2                           |
| Путь к data residency        | 4   | 2                    | 2                   | 0                           |
| Blast radius control         | 4   | 2                    | 2                   | 0                           |
| Масштабирование до 50 команд | 3   | 2                    | 1                   | 1                           |
| Итоговая взвешенная оценка   |     | 53                   | 31                  | 29                          |

Решение. Выбрана эволюционная стратегия: logical multi-tenancy на старте
и переход к cell-based архитектуре при росте числа tenant-ов или
появлении sensitive tenant-ов. Physical cluster per tenant отклонён
из-за стоимости и операционной сложности.

Положительные последствия:

- Низкая стоимость и сложность на старте.

- Сохраняется путь к усиленной изоляции через cells.

- Поддерживается data residency через размещение cells в нужных
  регионах.

Отрицательные последствия и меры контроля:

- На первом этапе blast radius выше; компенсируется квотами, rate
  limiting, security tests и meta-monitoring.

- Миграция tenant-а между cells требует отдельного runbook и
  предварительного тестирования.

### ADR-008. Выбор on-call и incident response подсистемы
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. On-call подсистема должна поддерживать расписания, escalation
chains, ack через mobile/external channel, SMS/voice и интеграцию с
Alertmanager.

| Критерий                             | Вес | Grafana OnCall + Twilio | PagerDuty | Opsgenie |
|--------------------------------------|-----|-------------------------|-----------|----------|
| Self-hosted / контроль critical path | 5   | 2                       | 0         | 0        |
| Mobile ack и escalation UX           | 4   | 1                       | 2         | 2        |
| Интеграция с Grafana/Alertmanager    | 4   | 2                       | 1         | 1        |
| Стоимость при росте пользователей    | 4   | 2                       | 0         | 1        |
| Зрелость incident management         | 3   | 1                       | 2         | 1        |
| Поддержка SMS/voice                  | 3   | 2                       | 2         | 2        |
| Итоговая взвешенная оценка           |     | 45                      | 30        | 28       |

Решение. Выбран Grafana OnCall self-hosted с внешним SMS/voice provider.
PagerDuty сохраняется как совместимая альтернатива для команд, у которых
уже есть внешний on-call процесс.

Положительные последствия:

- Сохраняется self-hosted модель и контроль над critical path.

- Нативная интеграция с выбранным observability stack.

- Снижается зависимость от per-user pricing.

Отрицательные последствия и меры контроля:

- Функции post-mortem и incident timeline слабее, чем у
  специализированных SaaS; закрываются Jira/Confluence-процессом.

- SMS/voice зависят от внешнего провайдера; нужен fallback и мониторинг
  доставки.

### ADR-009. Стратегия storage и retention
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. Storage является главным cost driver для observability.
Требуется раздельный retention по сигналам, возможность long-term
storage и восстановимость критичной конфигурации.

| Критерий                    | Вес | 2-tier hot + object storage | Single-tier block storage | 3-tier hot/warm/cold |
|-----------------------------|-----|-----------------------------|---------------------------|----------------------|
| Стоимость хранения          | 5   | 2                           | 0                         | 2                    |
| Операционная сложность      | 4   | 2                           | 2                         | 1                    |
| Query latency для hot data  | 4   | 2                           | 2                         | 2                    |
| Long retention              | 4   | 2                           | 1                         | 2                    |
| Поддержка Mimir/Loki/Tempo  | 4   | 2                           | 1                         | 1                    |
| Гибкость политики retention | 3   | 1                           | 1                         | 2                    |
| Итоговая взвешенная оценка  |     | 48                          | 25                        | 44                   |

Решение. Выбрана 2-tier стратегия: hot storage для оперативных данных и
S3-compatible object storage для холодного слоя, backup и immutable
audit artifacts. Трёхуровневая модель откладывается до появления
подтверждённого cost gap.

Положительные последствия:

- Баланс стоимости и операционной простоты.

- Поддерживается большинством выбранных компонентов.

- Упрощает DR и long-term retention.

Отрицательные последствия и меры контроля:

- Cold queries имеют большую latency; это явно отражается в NFR и UI как
  degraded/slow query mode.

- Object storage становится критичной зависимостью; требуется мониторинг
  доступности и backup credentials.

### ADR-010. GitOps и управление конфигурацией
Статус: Accepted. Decision-makers: архитектурная группа проекта,
SRE-команда, представитель product owner.

Контекст. Конфигурация СМА включает rules, dashboards, SLO, quotas,
RBAC, routing policies и on-call schedules. Ручное управление через UI
ведёт к drift и слабой воспроизводимости.

| Критерий                          | Вес | Argo CD + monorepo | Flux CD | UI-based manual config |
|-----------------------------------|-----|--------------------|---------|------------------------|
| Auditability через Git history    | 5   | 2                  | 2       | 0                      |
| Onboarding UX для команд          | 4   | 2                  | 1       | 2                      |
| Зрелость Kubernetes GitOps        | 4   | 2                  | 2       | 0                      |
| UI visibility sync state          | 3   | 2                  | 1       | 0                      |
| CODEOWNERS / path-based ownership | 3   | 2                  | 2       | 0                      |
| Риск configuration drift          | 3   | 2                  | 2       | 0                      |
| Итоговая взвешенная оценка        |     | 52                 | 45      | 8                      |

Решение. Выбран Argo CD с monorepo sma-config и path-based ownership.
UI-редактирование допускается только как prototyping path с последующим
экспортом в Git.

Положительные последствия:

- Единый источник истины для конфигурации.

- Воспроизводимость окружений и быстрый rollback через git revert.

- CI validation блокирует некорректные правила до production.

Отрицательные последствия и меры контроля:

- Входной порог для команд выше, чем у UI-only подхода; компенсируется
  шаблонами, onboarding wizard и документацией.

- Monorepo требует дисциплины CODEOWNERS и структуры каталогов.

## 4.12. Сводная таблица ADR
|         |                                           |                                       |                                         |
|---------|-------------------------------------------|---------------------------------------|-----------------------------------------|
| ID      | Решение                                   | Альтернативы (отвергнутые)            | Ключевой driver                         |
| ADR-001 | Grafana Mimir                             | Thanos, VictoriaMetrics, Cortex       | Multi-tenancy first-class + PromQL 100% |
| ADR-002 | Grafana Loki                              | OpenSearch, ClickHouse, Elasticsearch | Cost + correlation с метриками          |
| ADR-003 | Grafana Tempo                             | Jaeger, ClickHouse                    | Cost + native correlation               |
| ADR-004 | OTel Collector (agent+gateway)            | Vendor agents, Hybrid Fluent Bit      | C-TECH-03 + unified pipeline            |
| ADR-005 | Mimir Ruler + Alertmanager + Sloth        | Grafana Alerting unified, vmalert     | Native Mimir + proven at scale          |
| ADR-006 | Grafana OSS                               | Grafana Enterprise, Perses            | Maturity + AGPLv3                       |
| ADR-007 | Logical → Hybrid cell-based               | Physical per-tenant                   | Cost-fit на старте + path к isolation   |
| ADR-008 | Grafana OnCall + Twilio                   | PagerDuty, Opsgenie                   | Open source + no SaaS dependency        |
| ADR-009 | 2-tier (block + S3), per-signal retention | Single-tier, 3-tier                   | Cost balance + native support           |
| ADR-010 | Argo CD + monorepo GitOps                 | Flux, UI-based, hybrid                | Maturity + self-service UX              |
|         |                                           |                                       |                                         |

## 4.13. Архитектурная диаграмма компонентов
![Диаграмма](assets/image9.png)

### Краткое описание:
1.  Сбор данных  
    – OTel Collector Agent (DaemonSet на каждой ноде) собирает метрики,
    логи и трейсы с подов.  
    – OTel Collector Gateway (централизованный) выполняет
    tenant-роутинг, rate limiting и PII redaction, после чего направляет
    данные в соответствующие бэкенды.

2.  Хранение  
    – Mimir хранит метрики (15 дней на быстром диске, затем до 13
    месяцев в S3).  
    – Loki хранит логи (индексация по labels, данные в S3).  
    – Tempo хранит трейсы (полностью в S3 для экономии).

3.  Алертинг и on-call  
    – Mimir Ruler периодически вычисляет правила (PromQL).  
    – Alertmanager группирует, дедуплицирует, применяет silence и
    маршрутизирует.  
    – Notification Dispatcher обеспечивает доставку с повторами и
    fallback.  
    – Grafana OnCall управляет расписаниями, эскалацией и отправкой
    push/SMS/звонков через Twilio.

4.  Визуализация  
    – Grafana OSS предоставляет единый интерфейс для метрик, логов,
    трейсов, алертов и SLO.  
    – API Gateway открывает внешний REST/gRPC интерфейс для
    автоматизации.

5.  GitOps  
    – Вся конфигурация (правила, дашборды, SLO, tenants, RBAC,
    расписания) хранится в Git monorepo sma-config/.  
    – Argo CD синхронизирует желаемое состояние кластера с репозиторием.

## 4.14. Срабатывание алерта и эскалации (Сценарий 2)
## ![Диаграмма](assets/image10.png)

### Краткое описание:
1.  Сбор метрик – сервис отправляет метрики через OTel Agent и Gateway в
    Mimir. Gateway добавляет tenant-идентификатор и применяет rate
    limiting.

2.  Вычисление правил – Mimir Ruler каждые 30 секунд выполняет
    PromQL-правила. При превышении порога (например, error rate > 5% за
    5 минут) генерируется alert.

3.  Обработка алерта – Alertmanager группирует алерты, проверяет silence
    и inhibition, затем маршрутизирует уведомление в Notification
    Dispatcher.

4.  Эскалация и уведомления – Grafana OnCall последовательно отправляет
    уведомления: сначала в Slack и push, через 2 минуты – SMS, через 5
    минут – голосовой звонок. Если в течение 10 минут подтверждения нет
    – эскалация на тимлида.

5.  Подтверждение (ack) – инженер подтверждает алерт через мобильное
    приложение или Slack. Эскалация останавливается, статус алерта
    меняется на acked.

6.  Создание тикета – для P1-алертов автоматически создаётся инцидент в
    Jira (через outbound webhook).

## 4.15. Архитектура развертывания (High-level)
<span class="mark">Система мониторинга и алертинга разворачивается в
управляемом или self‑hosted Kubernetes (соответствие C‑TECH‑01). Все
компоненты упакованы в контейнеры Docker, оркестрируются через
Kubernetes и управляются декларативно с помощью GitOps (Argo CD,
ADR‑010). Ниже описано размещение, минимальное количество реплик для
отказоустойчивости и принципы масштабирования.</span>

### 4.15.1. Ключевые принципы
- Критичные компоненты (приём телеметрии, rules engine, alerting,
  notification dispatcher, API gateway) имеют не менее 3 реплик,
  распределённых по 3 availability zone (при облачном развёртывании).

- Stateful компоненты (Mimir ingester, Loki ingester, Tempo ingester)
  используют PersistentVolumeClaims для горячего блочного хранения и
  репликацию данных через внутренние механизмы (replication factor = 3).

- Холодное хранение (S3‑совместимый object storage) является общим для
  всех компонентов и обеспечивает durability на уровне провайдера (11
  девяток для AWS S3).

- Self‑monitoring и meta‑monitoring работают в отдельном неймспейсе и
  используют независимый канал уведомлений.

- Автомасштабирование включено для ingestion‑компонентов (OTel Gateway,
  Mimir distributor, Loki distributor) на основе CPU / memory / queue
  length.

### 4.15.2. Распределение по неймспейсам
|                   |                                                                       |
|-------------------|-----------------------------------------------------------------------|
| Namespace         | Компоненты                                                            |
| sma-collection    | OTel Agent (DaemonSet), OTel Gateway (Deployment)                     |
| sma-metrics       | Mimir distributor, ingester, querier, ruler, compactor, store‑gateway |
| sma-logs          | Loki distributor, ingester, querier, compactor                        |
| sma-traces        | Tempo distributor, ingester, querier, compactor                       |
| sma-alerting      | Alertmanager, Notification Dispatcher, Grafana OnCall                 |
| sma-visualization | Grafana OSS, API Gateway                                              |
| sma-gitops        | Argo CD (application controller, repo server, dex)                    |
| sma-backup        | Backup‑job, Velero (опционально)                                      |

**4.15.3. Начальное количество реплик и ресурсы (стартовый масштаб – до
20 tenant‑ов, ~500 сервисов)**

### Обоснование выбора начальных реплик и ресурсов
Приведенные в таблице значения являются стартовыми и базируются на:

- Рекомендациях производителей (Grafana Labs для Mimir/Loki/Tempo,
  OpenTelemetry) для production-конфигураций аналогичного масштаба.

- Оценке целевой нагрузки: ~1M samples/sec метрик, ~5–10 TB/сутки логов,
  ~100k spans/sec трейсов (согласно п. 1.4).

- Обеспечении отказоустойчивости: stateless компоненты – 3 реплики для
  переживания отказа AZ (QAS‑AVAIL‑02); stateful ingester’ы – RF=3 для
  durability (QAS‑RELI‑01).

- Запасе ёмкости (≈30% CPU/RAM) для пиковых нагрузок (QAS‑PERF‑02) и
  burst-сценариев.

- Ограничении C‑ORG‑01 (SRE-команда 5–7 человек) – начальная сложность
  ограничена, используется унифицированное управление через Helm.

После промышленной эксплуатации значения будут уточнены по результатам
нагрузочных тестов (TC‑05, TC‑26) и метрик реального потребления.
Автомасштабирование (HPA) позволит адаптироваться под рост без изменения
конфигурации.

|                         |        |             |                |                                  |
|-------------------------|--------|-------------|----------------|----------------------------------|
| Компонент               | Реплик | CPU request | Memory request | Примечание                       |
| OTel Gateway            | 3      | 2           | 4 GiB          | HPA до 10 при нагрузке           |
| Mimir distributor       | 3      | 2           | 4 GiB          | rate limiting + tenant routing   |
| Mimir ingester          | 3      | 8           | 32 GiB         | stateful, репликация RF=3        |
| Mimir querier           | 3      | 4           | 8 GiB          | горизонтально масштабируется     |
| Mimir ruler             | 2      | 2           | 4 GiB          | активно‑активный режим           |
| Loki distributor        | 3      | 1           | 2 GiB          | \-                               |
| Loki ingester           | 3      | 4           | 16 GiB         | stateful, RF=3                   |
| Loki querier            | 2      | 2           | 4 GiB          | \-                               |
| Tempo distributor       | 2      | 1           | 2 GiB          | \-                               |
| Tempo ingester          | 3      | 4           | 8 GiB          | stateful, RF=3                   |
| Tempo querier           | 2      | 2           | 4 GiB          | \-                               |
| Alertmanager            | 3      | 0.5         | 1 GiB          | HA конфигурация с шардированием  |
| Notification Dispatcher | 3      | 0.5         | 1 GiB          | stateless, HPA                   |
| Grafana OnCall          | 2      | 1           | 2 GiB          | база данных – отдельный Postgres |
| Grafana OSS             | 2      | 1           | 2 GiB          | shared database (PostgreSQL)     |
| API Gateway             | 3      | 1           | 2 GiB          | OAuth2 proxy + rate limiting     |

### 4.15.4. High Availability и отказоустойчивость
- Отказ одной availability zone (QAS‑AVAIL‑02):  
  Реплики компонентов распределены так, что при потере AZ оставшиеся две
  продолжают обслуживать трафик. Storage‑слой (Mimir, Loki, Tempo) имеет
  replication factor 3, поэтому потеря одной реплики не приводит к
  потере данных. Alerting pipeline переключается на живые реплики за <
  60 секунд.

- Отказ узла Kubernetes:  
  За счёт HPA и pod anti‑affinity поды автоматически пересоздаются на
  других узлах. StatefulSet с PVC могут мигрировать только при поддержке
  CSI‑драйвера с multi‑attach (например, Portworx) – в облаке это
  стандартная функциональность.

- Отказ компонента (QAS‑AVAIL‑01):  
  Приоритет отдаётся ingestion и alerting. Если выходит из строя Grafana
  или Query layer, метрики продолжают приниматься, правила продолжают
  вычисляться, уведомления доставляются. При деградации storage (логи
  или трейсы) система переходит в режим «minimal» – продолжает
  принимать, но запросы могут быть недоступны.

### 4.15.5. Сетевые политики и изоляция (security)
- Для каждого неймспейса настроены NetworkPolicy, разрешающие только
  необходимые ingress/egress.

- OTel Gateway и API Gateway доступны извне через Ingress Controller с
  TLS termination и OIDC аутентификацией (FR‑ADM‑004).

- Внутренние компоненты (Mimir ingester → Loki querier) общаются через
  mTLS или внутри кластерной сети с сетевыми политиками запрета по
  умолчанию.

### 4.15.6. Резервное копирование и DR (FR‑DR‑003, FR‑DR‑004)
- Конфигурация (rules, dashboards, SLO, tenants, RBAC, on‑call
  schedules) хранится в Git‑репозитории. Argo CD автоматически
  синхронизирует кластер. Дополнительный бэкап etcd не требуется.

- Состояние on‑call (расписания, переопределения) хранится в базе данных
  Grafana OnCall; её бэкап выполняется каждые 5 минут в S3 (RPO ≤ 5
  минут).

- Метрики, логи, трейсы уже дублируются в S3 через механизмы tiering.
  Для катастрофической потери региона планируется cross‑region
  репликация бакетов (опционально, для audit‑логов обязательно).

- DR drill проводится не реже 1 раза в квартал (TC‑23). Warm standby
  инстанс СМА в другом регионе может быть поднят из Git и бэкапов за <
  15 минут (RTO).

### 4.15.7. Визуализация развёртывания
![Диаграмма](assets/image11.png)

### Пояснение к диаграмме
- Три зоны доступности (AZ) – в каждой AZ развёрнуты реплики критических
  компонентов: OTel Gateway, Mimir distributor, Mimir ingester, Loki
  ingester, Alertmanager, API Gateway.

- Репликация данных – между ingester'ами (Mimir и Loki) настроена
  синхронная/асинхронная репликация с фактором репликации 3 (RF=3).
  Потеря одной AZ не приводит к потере данных.

- Общий холодный слой – все ingester'ы отправляют старые блоки/чанки в
  единый S3-совместимый object storage, доступный из любой AZ.

- Отказоустойчивость – при отказе одной AZ (например, AZ2) оставшиеся
  AZ1 и AZ3 продолжают приём телеметрии и обработку алертов.
  Alertmanager в режиме хаотичного шардирования перераспределяет
  нагрузку на живые реплики.

## 4.16. Восстановление СМА после сбоя (Сценарий 8)
Диаграмма показывает динамику failover с primary на secondary инстанс,
восстановление конфигурации и catch‑up уведомления.

![Диаграмма](assets/image12.png)

### Краткое описание:
1.  Обнаружение отказа – независимый meta‑monitoring (отдельный канал)
    не получает ответа от primary СМА в течение 5 минут и инициирует
    failover.

2.  Переключение DNS/Load Balancer – трафик направляется на warm standby
    (DR‑инстанс).

3.  Восстановление конфигурации – DR‑инстанс загружает:

    - rules, dashboards, SLO из Git‑репозитория;

    - on‑call schedules, silences, RBAC из бэкапа S3 (RPO ≤ 5 мин).

4.  Переподключение агентов – продуктовые сервисы автоматически
    переключаются на новый ingestion endpoint.

5.  Catch‑up уведомления – DR‑инстанс анализирует, какие алерты были
    сгенерированы во время недоступности primary, и отправляет сводку
    on‑call инженерам.

6.  Восстановление primary и failback – после восстановления основной
    системы выполняется контролируемый возврат трафика (обычно в окно
    технических работ).
