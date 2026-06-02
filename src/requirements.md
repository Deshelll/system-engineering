# 3. Требования и Quality Attributes
## 3.1. Назначение раздела
Раздел фиксирует требования к СМА на трёх уровнях:  
1. Stakeholder Requirements (STK) — потребности заинтересованных сторон;
описывает, что ожидают пользователи и бизнес.  
2. System Requirements / Functional Requirements (FR) — формализованные
требования к компонентам СМА, исходящие из сценариев CONOPS.  
3. Non-functional Requirements (NFR / Quality Attributes) —
нефункциональные требования с метриками, исходящие из QAS и constraints.

## 3.2. Stakeholder Requirements
|         |                             |                                                                                                       |                                                                                                       |                |
|---------|-----------------------------|-------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|----------------|
| ID      | Заинтересованная сторона    | Требование / Потребность                                                                              | Критерий успеха                                                                                       | Связь с CONOPS |
| STK-001 | SRE                         | Система должна своевременно обнаруживать деградации продуктовых сервисов                              | MTTD для P1/P2-инцидентов: p95 ≤ 5 минут                                                              | S2, S8         |
| STK-002 | On-call инженер             | Алерты должны быть релевантными и не создавать избыточной нагрузки на дежурного инженера              | False-positive rate ≤ 5%; signal-to-noise ≥ 80%                                                       | S2             |
| STK-003 | Разработчик / Service Owner | Подключение нового сервиса к мониторингу должно выполняться в self-service режиме                     | Time-to-first-metric ≤ 5 минут; полный onboarding p95 ≤ 1 час                                         | S1             |
| STK-004 | Тимлид                      | Система должна поддерживать управление on-call ротациями, подменами и эскалациями                     | Schedule coverage = 100%; swap применяется ≤ 1 минуты                                                 | S4             |
| STK-005 | Security инженер            | Система должна обеспечивать изоляцию tenant-ов, контроль доступа и защиту sensitive data в телеметрии | 100% попыток cross-tenant access блокируются; sensitive data redaction для стандартных шаблонов ≥ 95% | S3, S3, S5     |
| STK-006 | Platform team               | Система должна масштабироваться при росте количества сервисов и объёма телеметрии без нарушения SLA   | Масштабирование до 10x текущей нагрузки; деградация latency других tenant-ов ≤ 10%                    | S7             |
| STK-007 | Бизнес-владелец             | Стоимость эксплуатации observability-платформы должна быть контролируемой                             | Стоимость инфраструктуры СМА ≤ 3–5% от инфраструктурного бюджета наблюдаемых сервисов                 | S7             |
| STK-008 | Compliance                  | Критичные действия и конфигурация должны быть проверяемыми и восстанавливаемыми                       | Audit retention ≥ 2 года; RPO конфигурации ≤ 5 минут; RTO базовой функциональности ≤ 15 минут         | S8             |

## 3.3 STK → FR → NFR/QAS
| STK     | Потребность                                | Реализующие FR                                 | Поддерживающие NFR/QAS                           |
|---------|--------------------------------------------|------------------------------------------------|--------------------------------------------------|
| STK-001 | Своевременное обнаружение деградаций       | FR-ING-001, FR-RUL-001, FR-RUL-002, FR-NTF-001 | NFR-PERF-001, QAS-PERF-01                        |
| STK-002 | Снижение alert fatigue                     | FR-RUL-004, FR-RUL-005, FR-NTF-001, FR-SIL-001 | NFR-SCAL-001, QAS-SCAL-01                        |
| STK-003 | Self-service onboarding                    | FR-ING-005, FR-ADM-001, FR-UI-002, FR-RUL-001  | QAS-PERF-01                                      |
| STK-004 | Управление on-call                         | FR-OCM-001, FR-OCM-002, FR-NTF-003             | QAS-AVAIL-01                                     |
| STK-005 | Изоляция tenant-ов и защита sensitive data | FR-ING-006, FR-ADM-003, FR-ADM-005             | NFR-SEC-001, NFR-SEC-002, QAS-SEC-01, QAS-SEC-02 |
| STK-006 | Масштабирование                            | FR-ADM-002, FR-ING-004, FR-STO-002             | NFR-SCAL-001, NFR-SCAL-002                       |
| STK-007 | Контроль стоимости                         | FR-STO-001, FR-STO-002, FR-ADM-002             | C-COST-01, C-COST-02, C-COST-03                  |
| STK-008 | Audit, RPO/RTO, восстановление             | FR-ADM-005, FR-DR-001, FR-DR-003, FR-DR-004    | NFR-DR-001, QAS-MON-01                           |

## 3.4. System / Functional Requirements (FR)
Блок функциональных требований разбит по компонентам:  
- FR-ING — ingestion и приём телеметрии;  
- FR-STO — хранение данных;  
- FR-QRY — запросы и расследование;  
- FR-RUL — правила и генерация алертов;  
- FR-NTF — уведомления и маршрутизация;  
- FR-OCM — on-call management;

\- FR-SLO — SLO и error budget;

\- FR-SIL — silence и maintenance windows;

\- FR-API — API и интеграции;

\- FR-UI — пользовательский интерфейс;

\- FR-ADM — администрирование, multi-tenancy и RBAC;

\- FR-DR — disaster recovery и self-monitoring.

### 3.4.1. Ingestion (FR-ING)
|            |                   |                                                                                                                                                                       |                                                                                                                                             |           |                            |
|------------|-------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|-----------|----------------------------|
| ID         | Контекст (CONOPS) | Требование                                                                                                                                                            | Обоснование                                                                                                                                 | Приоритет | Проверка                   |
| FR-ING-001 | S1, S2            | Система MUST принимать метрики через OpenTelemetry Protocol (OTLP) и Prometheus Remote Write.                                                                         | Поддержка open-standard протоколов снижает зависимость от поставщика и обеспечивает совместимость с существующими Prometheus-инсталляциями. | MUST      | Integration test           |
| FR-ING-002 | S1, S3            | Система MUST принимать логи через OTLP Logs и syslog RFC 5424.                                                                                                        | Требование обеспечивает поддержку как современных сервисов, так и legacy-инфраструктуры.                                                    | MUST      | Integration test           |
| FR-ING-003 | S3                | Система MUST принимать distributed traces через OTLP Traces.                                                                                                          | Трейсы необходимы для анализа цепочек вызовов в микросервисной среде.                                                                       | MUST      | E2E trace propagation test |
| FR-ING-004 | S2, S6            | Ingestion pipeline MUST применять backpressure при превышении capacity и явно фиксировать отброшенные данные с указанием tenant_id, service_id, типа данных и объёма. | Молчаливая потеря данных недопустима, так как делает расследование инцидентов неполным.                                                     | MUST      | Load test 10x burst        |
| FR-ING-005 | S1                | Системе SHOULD автоматически обогащать телеметрию метаданными окружения: tenant_id, service_id, environment, namespace, pod, region.                                  | Автоматическое обогащение снижает количество ручной настройки при onboarding.                                                               | SHOULD    | E2E onboarding test        |
| FR-ING-006 | S1-S8             | Каждый ingestion-запрос MUST содержать tenant identifier; данные разных tenant-ов MUST быть логически изолированы на уровне ingestion, storage и query.               | Multi-tenancy является базовым требованием безопасности и масштабируемости.                                                                 | MUST      | Security test              |

## 3.4.2 Storage
| ID         | Требования                                                                                                                                                                                                | Приоритет | Обоснование                                                                 | Проверка                 |
|------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|-----------------------------------------------------------------------------|--------------------------|
| FR-STO-001 | Система MUST поддерживать раздельную настройку retention для метрик, логов, трейсов, алертов и audit log.                                                                                                 | MUST      | Разные типы данных имеют разную ценность и стоимость хранения.              | Configuration review     |
| FR-STO-002 | Система SHOULD поддерживать tiered storage: hot tier для часто используемых данных и object storage для холодных данных.                                                                                  | SHOULD    | Tiered storage снижает стоимость хранения при long-term retention.          | Demo + latency benchmark |
| FR-STO-003 | После подтверждения приёма данные MUST переживать отказ одного узла storage без потери.                                                                                                                   | MUST      | Потеря данных после acknowledgement нарушает доверие к системе мониторинга. | Chaos test               |
| FR-STO-004 | Метрики, логи, трейсы, алерты и audit log MUST быть защищены от несанкционированного изменения после записи. Удаление допускается только по retention policy или регламентированной compliance-процедуре. | MUST      | Неизменяемость данных необходима для расследований, аудита и post-mortem.   | Security audit           |

## 3.5 Query & Investigation
| ID         | Требования                                                                                                                                                                                            | Приоритет | Обоснование                                                                                     | Проверка                   |
|------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|-------------------------------------------------------------------------------------------------|----------------------------|
| FR-QRY-001 | Система MUST поддерживать PromQL или полностью совместимый язык запросов для метрик.                                                                                                                  | MUST      | PromQL является стандартным языком для SRE-сценариев, alerting rules и dashboards.              | Compatibility test         |
| FR-QRY-002 | Система MUST поддерживать фильтрацию и поиск логов по tenant_id, service_id, environment, level, time range и текстовому содержимому.                                                                 | MUST      | Логи должны быть пригодны для расследования инцидентов по сервису и временному окну.            | Functional test            |
| FR-QRY-003 | Пользователь MUST иметь возможность перейти из контекста алерта к связанным метрикам, логам и трейсам с сохранением tenant_id, service_id, environment и временного окна ±5 минут от alert timestamp. | MUST      | Сохранение контекста снижает время расследования и уменьшает риск ошибки при ручной фильтрации. | E2E investigation scenario |
| FR-QRY-004 | Пользователи SHOULD иметь возможность сохранять именованные запросы и передавать их другим пользователям через permalink.                                                                             | SHOULD    | Это упрощает эскалацию, handoff и подготовку post-mortem.                                       | Functional test            |

## 3.3.4 Rules & Alerts
<table>
<colgroup>
<col style="width: 13%" />
<col style="width: 24%" />
<col style="width: 17%" />
<col style="width: 26%" />
<col style="width: 17%" />
</colgroup>
<thead>
<tr class="header">
<th>ID</th>
<th>Требования</th>
<th>Приоритет</th>
<th>Обоснование</th>
<th>Проверка</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>FR-RUL-001</td>
<td>Alert rules MUST описываться декларативно в YAML или эквивалентном
формате, храниться в Git и применяться через CI/CD или API.</td>
<td>MUST</td>
<td><table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
</tbody>
</table>
<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>Rules-as-code обеспечивает версионирование, review и rollback.</th>
</tr>
</thead>
<tbody>
</tbody>
</table></td>
<td>GitOps validation test</td>
</tr>
<tr class="even">
<td>FR-RUL-002</td>
<td>Rules Engine MUST поддерживать составные условия AND/OR, временные
окна, агрегации по labels и задержку срабатывания вида for: 5m.</td>
<td>MUST</td>
<td>Составные условия снижают количество ложных срабатываний.</td>
<td>Functional test</td>
</tr>
<tr class="odd">
<td>FR-RUL-003</td>
<td>Система MUST поддерживать multi-window multi-burn-rate правила для
SLO-алертинга.</td>
<td>MUST</td>
<td>Burn-rate подход позволяет обнаруживать угрозу нарушения SLO раньше,
чем произойдёт полный breach.</td>
<td>Functional test</td>
</tr>
<tr class="even">
<td>FR-RUL-004</td>
<td>Система MUST группировать и дедуплицировать связанные алерты по
tenant_id, service_id, severity и environment.</td>
<td>MUST</td>
<td>Группировка предотвращает alert storm и снижает alert fatigue.</td>
<td>Scenario test</td>
</tr>
<tr class="odd">
<td>FR-RUL-005</td>
<td>Система SHOULD поддерживать inhibition rules, позволяющие подавлять
зависимые алерты при активном корневом алерте.</td>
<td>SHOULD</td>
<td>Это уменьшает шум при инфраструктурных сбоях.</td>
<td>Scenario test</td>
</tr>
</tbody>
</table>

## 3.3.5 Notification & Routing
<table>
<colgroup>
<col style="width: 17%" />
<col style="width: 21%" />
<col style="width: 19%" />
<col style="width: 22%" />
<col style="width: 18%" />
</colgroup>
<thead>
<tr class="header">
<th>ID</th>
<th>Требования</th>
<th>Приоритет</th>
<th>Обоснование</th>
<th>Проверка</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>FR-NTF-001</td>
<td><table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
</tbody>
</table>
<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>Routing policies MUST определять получателей алерта на основе
tenant_id, service_id, owner_team, severity, environment и времени
суток.</th>
</tr>
</thead>
<tbody>
</tbody>
</table></td>
<td>MUST</td>
<td>Алерт должен попадать к ответственному инженеру или команде.</td>
<td>Routing matrix test</td>
</tr>
<tr class="even">
<td>FR-NTF-002</td>
<td>Система MUST поддерживать доставку уведомлений через chat-каналы,
email, mobile push, SMS, voice call и webhook.</td>
<td>MUST</td>
<td>Для разных уровней критичности требуются разные каналы
доставки.</td>
<td>Integration test</td>
</tr>
<tr class="odd">
<td>FR-NTF-003</td>
<td>Система MUST поддерживать эскалационные цепочки: primary on-call →
secondary on-call → team lead / manager.</td>
<td>MUST</td>
<td>Критичный алерт не должен оставаться без ответственного лица.</td>
<td>No-ack scenario test</td>
</tr>
<tr class="even">
<td>FR-NTF-004</td>
<td>Получатель MUST иметь возможность подтвердить алерт через UI,
mobile-friendly интерфейс или внешний канал без обязательного перехода в
основной UI.</td>
<td>MUST</td>
<td>Быстрый ack уменьшает Time-to-Acknowledge.</td>
<td>Usability test</td>
</tr>
<tr class="odd">
<td>FR-NTF-005</td>
<td>Notification pipeline MUST обеспечивать retry с backoff; После 5
неуспешных попыток доставки в течение 10 минут событие MUST попадать в
dead-letter queue. Каждая попытка доставки MUST фиксироваться с
указанием receiver_id, channel, timestamp, error_code и
retry_attempt.</td>
<td>MUST</td>
<td>Ошибка канала доставки не должна приводить к бесследной потере
критичного алерта.</td>
<td>Chaos test</td>
</tr>
</tbody>
</table>

## 3.3.6 On-call Management
<table>
<colgroup>
<col style="width: 17%" />
<col style="width: 22%" />
<col style="width: 18%" />
<col style="width: 22%" />
<col style="width: 18%" />
</colgroup>
<thead>
<tr class="header">
<th>ID</th>
<th>Требования</th>
<th>Приоритет</th>
<th>Обоснование</th>
<th>Проверка</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>FR-OCM-001</td>
<td><table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
</tbody>
</table>
<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>Система MUST поддерживать создание on-call расписаний с primary и
secondary responder.</th>
</tr>
</thead>
<tbody>
</tbody>
</table></td>
<td>MUST</td>
<td>Расписание необходимо для маршрутизации алертов.</td>
<td>Functional test</td>
</tr>
<tr class="even">
<td>FR-OCM-002</td>
<td>Система MUST поддерживать swap и override дежурств; изменение MUST
применяться к маршрутизации в течение 1 минуты и фиксироваться в audit
log.</td>
<td>MUST</td>
<td>Подмены должны сразу влиять на доставку алертов.</td>
<td>Scenario test</td>
</tr>
<tr class="odd">
<td>FR-OCM-003</td>
<td>Система SHOULD поддерживать экспорт on-call расписаний в iCal /
Google Calendar.</td>
<td>SHOULD</td>
<td>Интеграция с календарями повышает удобство эксплуатации.</td>
<td>Demo</td>
</tr>
</tbody>
</table>

## 3.3.7 SLO & Error Budget
<table>
<colgroup>
<col style="width: 14%" />
<col style="width: 20%" />
<col style="width: 18%" />
<col style="width: 28%" />
<col style="width: 17%" />
</colgroup>
<thead>
<tr class="header">
<th>ID</th>
<th>Требования</th>
<th>Приоритет</th>
<th>Обоснование</th>
<th>Проверка</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>FR-SLO-001</td>
<td>SLO MUST описываться декларативно с указанием service_id, SLI,
target и rolling window.</td>
<td>MUST</td>
<td><table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
</tbody>
</table>
<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>Декларативное описание делает SLO воспроизводимыми и
проверяемыми.</th>
</tr>
</thead>
<tbody>
</tbody>
</table></td>
<td>API / GitOps review</td>
</tr>
<tr class="even">
<td>FR-SLO-002</td>
<td><table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
</tbody>
</table>
<table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<thead>
<tr class="header">
<th>Система MUST непрерывно вычислять текущий SLI, оставшийся error
budget и burn rate.</th>
</tr>
</thead>
<tbody>
</tbody>
</table></td>
<td>MUST</td>
<td>Эти показатели необходимы для оценки надёжности сервиса.</td>
<td>Functional test</td>
</tr>
<tr class="odd">
<td>FR-SLO-003</td>
<td>Система MUST предоставлять SLO dashboard по tenant-у и сервису.</td>
<td>MUST</td>
<td>Команды должны видеть состояние SLO и причины расходования error
budget.</td>
<td>UI review</td>
</tr>
</tbody>
</table>

## 3.3.8 Silence & Maintenance
| ID         | Требования                                                                                                                                       | Приоритет | Обоснование                                                            | Проверка             |
|------------|--------------------------------------------------------------------------------------------------------------------------------------------------|-----------|------------------------------------------------------------------------|----------------------|
| FR-SIL-001 | Silence MUST содержать matcher, start_time, end_time, автора и обязательную причину.                                                             | MUST      | Silence без причины затрудняет аудит и может скрыть реальный инцидент. | Form validation test |
| FR-SIL-002 | Silence MUST автоматически истекать по заданному времени; максимальная длительность без продления — 7 дней.                                      | MUST      | Автоматическое истечение предотвращает “забытые” silence.              | Functional test      |
| FR-SIL-003 | Silence, покрывающий env=prod или более 10 production-сервисов, MUST требовать подтверждения второго инженера с ролью SRE Lead или Tenant Admin. | SHOULD    | Это снижает риск случайного подавления критичных алертов.              | Scenario test        |

## 3.3.9 API & Integration
<table>
<colgroup>
<col style="width: 13%" />
<col style="width: 28%" />
<col style="width: 17%" />
<col style="width: 22%" />
<col style="width: 17%" />
</colgroup>
<thead>
<tr class="header">
<th>ID</th>
<th>Требования</th>
<th>Приоритет</th>
<th>Обоснование</th>
<th>Проверка</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>FR-API-001</td>
<td>Все основные операции, доступные через UI, MUST быть доступны через
документированный API.</td>
<td>MUST</td>
<td><table>
<colgroup>
<col style="width: 100%" />
</colgroup>
<tbody>
</tbody>
</table>
<p>API необходим для автоматизации, GitOps и интеграций.</p></td>
<td>OpenAPI review</td>
</tr>
<tr class="even">
<td>FR-API-002</td>
<td>Система MUST поддерживать outbound webhooks для событий alert
created, alert acked, alert resolved, silence created, SLO breach.</td>
<td>MUST</td>
<td>Webhook-события нужны для интеграции с внешними процессами и
системами.</td>
<td>Integration test</td>
</tr>
<tr class="odd">
<td>FR-API-003</td>
<td>Система SHOULD предоставлять готовые интеграции с Jira, ServiceNow,
Slack, Microsoft Teams и PagerDuty-compatible endpoint.</td>
<td>SHOULD</td>
<td>Готовые интеграции сокращают трудоёмкость внедрения.</td>
<td>Per-integration test</td>
</tr>
</tbody>
</table>

## 3.3.10 User Interface
| ID        | Требования                                                                                                                            | Приоритет | Обоснование                                                           | Проверка                   |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------|-----------|-----------------------------------------------------------------------|----------------------------|
| FR-UI-001 | Метрики, логи, трейсы, алерты и SLO MUST быть доступны в едином UI с сохранением контекста tenant, service, environment и time range. | MUST      | Единый UI снижает переключение между инструментами при расследовании. | E2E investigation scenario |
| FR-UI-002 | Dashboards SHOULD описываться декларативно и версионироваться в Git; UI-редактор SHOULD поддерживать экспорт определения dashboard.   | SHOULD    | Dashboards-as-code повышает воспроизводимость и снижает drift.        | GitOps demo                |
| FR-UI-003 | Любое состояние UI, включая запрос, фильтры, временное окно и выбранную панель, MUST быть представимо permalink-ссылкой.              | MUST      | Permalink нужен для эскалации и передачи контекста.                   | Functional test            |
| FR-UI-004 | Базовые UI-операции MUST завершаться за p95 ≤ 3 секунды при штатной нагрузке.                                                         | MUST      | Медленный UI увеличивает время расследования инцидента.               | UI performance test        |
| FR-UI-005 | Система SHOULD предоставлять mobile-friendly интерфейс для ack алерта, просмотра краткого контекста и запуска эскалации.              | SHOULD    | On-call инженер часто реагирует на алерт вне рабочего места.          | Mobile usability test      |

## 3.3.11 Administration & Multi-tenancy
| ID         | Требования                                                                                                             | Приоритет | Обоснование                                                                | Проверка               |
|------------|------------------------------------------------------------------------------------------------------------------------|-----------|----------------------------------------------------------------------------|------------------------|
| FR-ADM-001 | Система MUST поддерживать lifecycle tenant-а: create, configure quotas, suspend, archive, delete.                      | MUST      | Tenant lifecycle необходим для self-service подключения команд.            | API test               |
| FR-ADM-002 | Для каждого tenant-а MUST задаваться квоты на ingestion rate, storage, cardinality и query concurrency.                | MUST      | Квоты защищают платформу от noisy neighbor и cost overrun.                 | Load & isolation test  |
| FR-ADM-003 | Система MUST поддерживать RBAC с ролями Viewer, Editor, On-call Responder, Tenant Admin и Platform Admin.              | MUST      | RBAC обеспечивает разграничение доступа между пользователями и tenant-ами. | Permission matrix test |
| FR-ADM-004 | Аутентификация MUST поддерживать OIDC / SAML 2.0; локальные учётные записи допускаются только для break-glass доступа. | MUST      | Интеграция с корпоративным IdP необходима для управления доступом.         | Integration test       |
| FR-ADM-005 | Все критичные действия пользователей MUST фиксироваться в immutable audit log с retention ≥ 2 года.                    | MUST      | Audit log необходим для compliance, расследований и контроля изменений.    | Audit log review       |

## 3.3.12 Disaster Recovery & Self-monitoring
| ID        | Требования                                                                                                                                              | Приоритет | Обоснование                                                           | Проверка        |
|-----------|---------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|-----------------------------------------------------------------------|-----------------|
| FR-DR-001 | СМА MUST мониториться независимым meta-monitoring контуром с отдельным notification-каналом.                                                            | MUST      | Если основная СМА недоступна, отказ должен быть обнаружен независимо. | Chaos test      |
| FR-DR-002 | При частичном отказе система MUST деградировать по приоритетам: alerting и ingestion метрик сохраняются в первую очередь; UI/query могут деградировать. | MUST      | В инциденте приоритет имеет обнаружение и доставка алертов.           | Chaos test      |
| FR-DR-003 | Конфигурация rules, dashboards, SLO, schedules, tenants и RBAC MUST регулярно бэкапиться с RPO ≤ 5 минут.                                               | MUST      | Потеря конфигурации делает восстановление системы неполным.           | Restore drill   |
| FR-DR-004 | MUST существовать проверенный runbook восстановления СМА; DR drill проводится не реже 1 раза в квартал.                                                 | MUST      | Непроверенная процедура восстановления не гарантирует достижение RTO. | DR drill report |

## 3.4. Non-Functional Requirements / Quality Attributes (NFR)
|               |                   |                                                                                |                                                                                                                                                     |            |                   |
|---------------|-------------------|--------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|------------|-------------------|
| ID            | Категория         | Требование                                                                     | Метрика                                                                                                                                             | Контекст   | Verification      |
| NFR-PERF-001  | Performance       | Ingestion pipeline должен обрабатывать штатную нагрузку без значимой задержки. | При нагрузке 1M samples/sec, 500K log entries/sec и 100K spans/sec: ack latency p95 ≤ 1 c, p99 ≤ 3 c; end-to-end visibility ≤ 30 c.                 | S1, S2     | Load test         |
| NFR-PERF-002  | Performance       | Система должна выдерживать burst-нагрузку без каскадного отказа.               | При 10x burst в течение 5 минут: data loss для принятых метрик = 0; низкоприоритетные логи могут ограничиваться явно через backpressure/rate limit. | S2, S7     | Load test         |
| NFR-PERF-003  | Performance       | Query latency должна позволять оперативное расследование инцидента.            | Метрики за 1 час: p95 ≤ 2 c; логи за 1 час: p95 ≤ 5 c; trace by ID: p95 ≤ 1 c.                                                                      | S3         | Latency benchmark |
| NFR-AVAIL-001 | Availability      | Alerting path должен оставаться доступным при отказе UI или query layer.       | Availability alerting pipeline ≥ 99.95% за календарный месяц, исключая согласованные maintenance windows.                                           | S2, S8     | Chaos test        |
| NFR-AVAIL-002 | Availability      | Система должна переживать отказ одной availability zone.                       | Потеря данных = 0; alerting downtime ≤ 60 c; full functionality recovery ≤ 5 минут.                                                                 | S8         | AZ failure test   |
| NFR-RELI-001  | Reliability       | Данные не должны теряться после acknowledgement приёма.                        | Durability ≥ 99.99% для метрик, логов и трейсов; для алертов и audit log — 100% логическая сохранность подтверждённых событий.                      | S3, S8     | Chaos test        |
| NFR-SCAL-001  | Scalability       | Один tenant не должен деградировать работу других tenant-ов.                   | При превышении tenant-ом квоты в 100x latency других tenant-ов не должна превышать baseline более чем на 10%.                                       | S7         | Isolation test    |
| NFR-SCAL-002  | Scalability       | Система должна горизонтально масштабироваться при росте нагрузки.              | Поддержка роста до 10x текущей нагрузки с efficiency ≥ 70% от линейного масштабирования.                                                            | S7         | Scalability test  |
| NFR-SEC-001   | Security          | Cross-tenant access должен блокироваться.                                      | 100% попыток доступа к данным другого tenant-а возвращают 403 и фиксируются в audit log.                                                            | S1, S3, S5 | Security test     |
| NFR-SEC-002   | Security          | Sensitive data в логах и трейсах должны редактироваться до записи.             | Detection/redaction стандартных шаблонов PII/secrets ≥ 95%; redaction latency ≤ 10 ms на log entry.                                                 | S3         | Security audit    |
| NFR-DR-001    | Disaster Recovery | Базовая функциональность СМА должна восстанавливаться после серьёзного сбоя.   | RTO ≤ 15 минут; RPO конфигурации ≤ 5 минут; DR drill ≥ 1 раз в квартал.                                                                             | S8         | DR drill          |

## 3.5. Формат и нотация
### 3.5.1 Quality Attribute Scenarios
| ID           | Атрибут качества         | Сценарий                                                                 | Измеримый результат                                                                              | Связанные требования                            |
|--------------|--------------------------|--------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|-------------------------------------------------|
| QAS-PERF-01  | Performance              | Продуктовые сервисы передают штатный поток телеметрии в production.      | Ack latency p95 ≤ 1 c; p99 ≤ 3 c; данные доступны для query ≤ 30 c.                              | FR-ING-001–004, NFR-PERF-001                    |
| QAS-PERF-02  | Performance / Resilience | Один tenant генерирует 10x burst телеметрии в течение 5 минут.           | Backpressure активируется; принятые метрики не теряются; система не уходит в каскадный отказ.    | FR-ING-004, FR-ADM-002, NFR-PERF-002            |
| QAS-PERF-03  | Performance              | On-call инженер выполняет расследовательский запрос в UI.                | Метрики p95 ≤ 2 c; логи p95 ≤ 5 c; trace by ID p95 ≤ 1 c.                                        | FR-QRY-001–003, FR-UI-004, NFR-PERF-003         |
| QAS-AVAIL-01 | Availability             | Отказывает UI или query layer.                                           | Alerting и ingestion метрик продолжают работать; деградация явно отображается в self-monitoring. | FR-DR-001, FR-DR-002, NFR-AVAIL-001             |
| QAS-AVAIL-02 | Availability             | Отказывает одна availability zone.                                       | Потеря данных = 0; alerting downtime ≤ 60 c; восстановление полной функциональности ≤ 5 минут.   | FR-STO-003, FR-DR-002, NFR-AVAIL-002            |
| QAS-RELI-01  | Reliability              | После ack приёма происходит отказ storage-узла.                          | Подтверждённые данные доступны после отказа; corruption восстанавливается из реплик.             | FR-STO-003, NFR-RELI-001                        |
| QAS-SCAL-01  | Scalability              | Tenant превышает квоту в 100x из-за log storm или cardinality explosion. | Tenant ограничивается по квоте; другие tenant-ы не получают деградацию latency более 10%.        | FR-ADM-002, FR-ING-006, NFR-SCAL-001            |
| QAS-SCAL-02  | Scalability              | Нагрузка платформы постепенно растёт до 10x.                             | Система масштабируется горизонтально без изменения архитектуры и без downtime.                   | FR-ADM-002, NFR-SCAL-002                        |
| QAS-SEC-01   | Security                 | Пользователь tenant A пытается получить данные tenant B.                 | Запрос отклоняется с 403; попытка фиксируется в audit log.                                       | FR-ING-006, FR-ADM-003, FR-ADM-005, NFR-SEC-001 |
| QAS-SEC-02   | Security                 | В лог попадает sensitive payload.                                        | Sensitive data редактируются до записи; событие фиксируется без сохранения исходного секрета.    | FR-ING-002, FR-ADM-005, NFR-SEC-002             |
| QAS-MON-01   | Self-monitoring          | Компонент СМА деградирует или становится недоступен.                     | Meta-monitoring обнаруживает отказ ≤ 2 минут и уведомляет через независимый канал.               | FR-DR-001, NFR-DR-001                           |

### Формат FR
**FR-XXX Краткое имя требования** ─────────────────────────────────────

Контекст: ссылка на сценарий CONOPS (S1–S8) или общесистемное
Требование: формулировка с MUST/SHOULD/MAY (RFC 2119). Перевод: MUST =
"ДОЛЖНА"; SHOULD = "СЛЕДУЕТ" / "РЕКОМЕНДУЕТСЯ"; MAY = "МОЖЕТ" Rationale:
зачем это нужно, какую боль решает Приоритет: MUST \| SHOULD \| MAY
Verification: способ проверки (test, review, demo, analysis)

### Уровни приоритета (RFC 2119)
|         |                            |                                          |
|---------|----------------------------|------------------------------------------|
| Уровень | Значение                   | Что будет, если не реализовать           |
| MUST    | Обязательное требование    | система не выполняет основное назначение |
| SHOULD  | Настоятельно рекомендуется | деградация UX или операционная боль      |
| MAY     | Опциональное               | можно отложить без последствий           |

Группировка FR

Требования сгруппированы по функциональным областям, соответствующим
компонентам будущей декомпозиции (Раздел 5):

- FR-ING-xxx — Ingestion (приём метрик/логов/трейсов)

- FR-STO-xxx — Storage (хранение, retention)

- FR-QRY-xxx — Query & Investigation (поиск, корреляция)

- FR-RUL-xxx — Rules & Alerts (правила, генерация алертов)

- FR-NTF-xxx — Notification & Routing (доставка, эскалация)

- FR-OCM-xxx — On-call Management (расписания, дежурства)

- FR-SLO-xxx — SLO & Error Budget

- FR-SIL-xxx — Silence & Maintenance

- FR-API-xxx — API & Integration

- FR-UI-xxx — User Interface

- FR-ADM-xxx — Administration & Multi-tenancy

- FR-DR-xxx — Disaster Recovery & Self-monitoring

## 3.6. Детализированные карточки требований
### 3.6.1. Ingestion (FR-ING)
### FR-ING-001 Приём метрик через open-standard протоколы
───────────────────────────────────────────── Контекст: S1 (onboarding),
S2 (детекция)

Требование: Система ДОЛЖНА (MUST) принимать метрики через OpenTelemetry
Protocol (OTLP) и Prometheus Remote Write. Rationale: Open standards
снижают vendor lock-in; OTLP — стратегический стандарт CNCF; Prometheus
RW — де-факто стандарт в SRE.

Приоритет: MUST Verifica\tion: Integration test с эталонным OTel
collector и Prometheus

### FR-ING-002 Приём логов через open-standard протоколы
───────────────────────────────────────────── Контекст: S1, S3
(расследование)

Требование: Система ДОЛЖНА (MUST) принимать логи через OTLP Logs и
syslog (RFC 5424). Rationale: OTLP для современных сервисов, syslog —
для legacy и инфраструктурных компонентов.

Приоритет: MUST Verification: Integration test

### FR-ING-003 Приём трейсов через OTLP
───────────────────────────────────────────── Контекст: S3
(расследование, distributed tracing)

Требование: Система ДОЛЖНА (MUST) принимать distributed traces через
OTLP Traces. Rationale: Трейсы критичны для расследования в
микросервисной среде; OTLP — единственный жизнеспособный стандарт.
Приоритет: MUST Verification: Integration test, end-to-end trace
propagation check

### FR-ING-004 Backpressure и устойчивость к burst
───────────────────────────────────────────── Контекст: общесистемное;
связано с QAS-PERF-01

Требование: Ingestion pipeline ДОЛЖЕН (MUST) применять backpressure при
превышении capacity, а не терять данные молча. Отброшенные данные ДОЛЖНЫ
(MUST) логироваться с указанием источника и объёма. Rationale:
Молчаливая потеря данных подрывает доверие к системе и делает
невозможным расследование.

Приоритет: MUST Verification: Load test с 10x burst, проверка metrics о
drop rate

### FR-ING-005 Авто-обнаружение метаданных
───────────────────────────────────────────── Контекст: S1 (onboarding),
снижение ручной конфигурации

Требование: Система ДОЛЖНА (SHOULD) автоматически обогащать входящую
телеметрию метаданными окружения (k8s namespace, pod, cloud region) на
основе resource attributes OTel. Rationale: Снижает ошибки конфигурации,
упрощает onboarding (S1 цель < 1 рабочего дня).

Приоритет: SHOULD Verification: E2E test onboarding-сценария

### FR-ING-006 Tenant isolation на уровне ingestion
───────────────────────────────────────────── Контекст: общесистемное;
multi-tenancy

Требование: Каждый ingestion-запрос ДОЛЖЕН (MUST) содержать tenant
identifier; данные разных тенантов ДОЛЖНЫ (MUST) быть логически
изолированы. Rationale: Multi-tenancy — базовое требование платформы;
без изоляции невозможны RBAC и data governance.

Приоритет: MUST Verification: Security test, попытка cross-tenant access

### 3.6.2. Storage (FR-STO)
### FR-STO-001 Дифференцированный retention по типам данных
───────────────────────────────────────────── Контекст: общесистемное;
стоимость vs доступность

Требование: Система ДОЛЖНА (MUST) поддерживать конфигурируемый retention
отдельно для метрик, логов, трейсов и алертов. Минимум: метрики 13
месяцев (year-over-year), логи 30 дней hot + 90 дней warm, трейсы 7
дней, алерты 2 года. Rationale: Разные данные имеют разную ценность во
времени; единый retention либо разоряет (всё в hot), либо ломает
расследование. Приоритет: MUST Verification: Конфигурационный review +
retention test

### FR-STO-002 Tiered storage
───────────────────────────────────────────── Контекст: cost-efficiency,
S3 расследование старых инцидентов

Требование: Система ДОЛЖНА (SHOULD) автоматически перемещать «холодные»
данные в object storage (E6) с сохранением возможности query (с
допустимой деградацией latency). Rationale: Без tiering стоимость
хранения растёт линейно с retention; S3 хранение в 10–50x дешевле block
storage. Приоритет: SHOULD Verification: Demo: query 6-месячных метрик,
latency benchmark

### FR-STO-003 Durability данных
───────────────────────────────────────────── Контекст: S8 (DR),
compliance

Требование: После подтверждения приёма данные ДОЛЖНЫ (MUST) переживать
отказ одного узла без потери. Целевая durability >= 99.99% для метрик и
логов, 100% для алертов. Rationale: Потеря данных мониторинга = потеря
доказательной базы для post-mortem и audit.

Приоритет: MUST Verification: Chaos test (kill node), data integrity
check

### FR-STO-004 Запрет на изменение исторических данных
───────────────────────────────────────────── Контекст: compliance,
аудит

Требование: Метрики, логи и трейсы ДОЛЖНЫ (MUST) быть immutable после
записи (append-only). Удаление допустимо только по retention policy или
GDPR-запросу с записью в audit log. Rationale: Immutability — базовое
требование для доверия к данным расследования и для compliance.
Приоритет: MUST Verification: API review, security audit

### 3.6.3. Query & Investigation (FR-QRY)
### FR-QRY-001 Query language для метрик
───────────────────────────────────────────── Контекст: S3
(расследование), S6 (SLO)

Требование: Система ДОЛЖНА (MUST) поддерживать PromQL или совместимый
язык запросов для метрик. Rationale: PromQL — де-факто стандарт;
экосистема dashboards (Grafana) и алертов на нём построена. Приоритет:
MUST Verification: Compatibility test с эталонным PromQL test suite

### FR-QRY-002 Query language для логов
───────────────────────────────────────────── Контекст: S3

Требование: Система ДОЛЖНА (MUST) поддерживать структурный язык запросов
для логов (LogQL-совместимый или эквивалент) с поддержкой full-text
search, label filtering и агрегаций.

Приоритет: MUST Verification: Functional test

### FR-QRY-003 Кросс-сигнальная корреляция
───────────────────────────────────────────── Контекст: S3 (главная боль
расследования)

Требование: Из контекста алерта или метрики пользователь ДОЛЖЕН (MUST)
иметь возможность одним кликом перейти к соответствующим логам и трейсам
того же сервиса и временного окна. Rationale: Корреляция
metrics→logs→traces — главный value-driver observability; ручное
переключение между инструментами — основная причина высокого MTTR.
Приоритет: MUST Verification: Usability test, E2E investigation scenario

### FR-QRY-004 Сохранение и шеринг запросов
──────────────────────────────────────────── Контекст: S3, накопление
institutional knowledge

Требование: Пользователи ДОЛЖНЫ (SHOULD) иметь возможность сохранять
named queries и делиться ими через permalink. Rationale: Snapshot
состояния расследования критичен для передачи контекста при эскалации и
для post-mortem.

Приоритет: SHOULD Verification: Functional test

### 3.3.4. Rules & Alerts (FR-RUL)
### FR-RUL-001 Декларативное описание правил (rules as code)
───────────────────────────────────────────── Контекст: S1, S6,
общесистемное

Требование: Alert rules ДОЛЖНЫ (MUST) описываться декларативно (YAML или
эквивалент), храниться в Git, применяться через API/CI. Rationale: Rules
as code = версионирование, code review, rollback, reproducibility.
UI-only configuration ведёт к drift и непроверяемым изменениям.

Приоритет: MUST Verification: API review, CI integration demo

### FR-RUL-002 Multi-condition правила
───────────────────────────────────────────── Контекст: S2, снижение
false-positive

Требование: Правила ДОЛЖНЫ (MUST) поддерживать составные условия
(AND/OR), временные окна (for: 5m), агрегации по labels. Rationale:
Простые threshold-алерты дают высокий false-positive rate;
multi-condition позволяет точнее ловить реальные проблемы.

Приоритет: MUST Verification: Functional test

### FR-RUL-003 Multi-window multi-burn-rate для SLO
───────────────────────────────────────────── Контекст: S6 (SLO)

Требование: Система ДОЛЖНА (MUST) поддерживать burn-rate алерты с
несколькими окнами (fast/slow burn) согласно Google SRE Workbook
методологии. Rationale: Burn-rate — лучшая практика для SLO-алертинга;
single threshold даёт либо ложные срабатывания, либо поздние.

Приоритет: MUST Verification: Functional test, документация методологии

### FR-RUL-004 Группировка и дедупликация алертов
───────────────────────────────────────────── Контекст: S2, alert
fatigue prevention

Требование: Система ДОЛЖНА (MUST) группировать связанные алерты по
конфигурируемым labels и подавлять дубликаты в пределах окна. Rationale:
Без группировки один инцидент может породить сотни notifications —
главная причина alert fatigue.

Приоритет: MUST Verification: Scenario test: один инцидент → одна группа
notifications

### FR-RUL-005 Подавление зависимых алертов (inhibition)
───────────────────────────────────────────── Контекст: S2

Требование: Система ДОЛЖНА (SHOULD) поддерживать правила inhibition
(если алерт A активен, подавить алерт B). Rationale: Если упал кластер
целиком — не нужны 200 алертов про отдельные сервисы; нужен один
корневой.

Приоритет: SHOULD Verification: Scenario test

### 3.3.5. Notification & Routing (FR-NTF)
### FR-NTF-001 Маршрутизация по policy
───────────────────────────────────────────── Контекст: S2

Требование: Маршрутизация алертов ДОЛЖНА (MUST) определяться
декларативными routing policies на основе labels (team, severity, env) и
времени (рабочие часы / on-call).

Приоритет: MUST Verification: Routing policy test matrix

### FR-NTF-002 Множественные каналы доставки
───────────────────────────────────────────── Контекст: S2

Требование: Система ДОЛЖНА (MUST) поддерживать каналы: Slack/Teams,
email, SMS, voice call, mobile push, webhook. Канал выбирается на основе
severity и policy. Rationale: P1 требует push/voice (гарантия pickup),
P3 — chat (не будит ночью). Single-channel система непригодна.

Приоритет: MUST Verification: Integration test per channel

### FR-NTF-003 Эскалационные цепочки
───────────────────────────────────────────── Контекст: S2 (шаги 7–8)

Требование: Система ДОЛЖНА (MUST) поддерживать эскалационные цепочки:
если primary on-call не подтвердил алерт за N минут — эскалация на
secondary, далее на manager. Rationale: Гарантия pickup критичных
алертов; без эскалации P1 может висеть всю ночь.

Приоритет: MUST Verification: Scenario test с симуляцией no-ack

### FR-NTF-004 Acknowledgement и lifecycle алерта
───────────────────────────────────────────── Контекст: S2

Требование: Получатель ДОЛЖЕН (MUST) иметь возможность подтвердить (ack)
или передать (re-route) алерт через primary канал без перехода в UI.
Lifecycle: triggered → acked → resolved/expired. Rationale: Минимизация
trips в UI критична для MTTR; ack через Slack экономит 30–60 сек на
каждом алерте.

Приоритет: MUST Verification: Usability test

### FR-NTF-005 Гарантия доставки (at-least-once)
───────────────────────────────────────────── Контекст: S2, S8

Требование: Notification pipeline ДОЛЖЕН (MUST) обеспечивать
at-least-once доставку с retry. Недоставленные после N попыток алерты
ДОЛЖНЫ (MUST) попадать в dead-letter queue с alert о факте недоставки.

Приоритет: MUST Verification: Chaos test (kill channel), DLQ check

### 3.3.6. On-call Management (FR-OCM)
### FR-OCM-001 Декларативные расписания
───────────────────────────────────────────── Контекст: S4 Требование:
Расписания дежурств ДОЛЖНЫ (MUST) описываться декларативно (ротации,
override, holiday calendars) и храниться в Git или managed UI с
экспортом.

Приоритет: MUST Verification: API review

### FR-OCM-002 Подмены и handoff
───────────────────────────────────────────── Контекст: S4

Требование: Инженеры ДОЛЖНЫ (MUST) иметь возможность инициировать
подмену; подмена ДОЛЖНА (MUST) фиксироваться в audit log и применяться к
маршрутизации в течение 1 минуты.

Приоритет: MUST Verification: Scenario test

### FR-OCM-003 Calendar integration
───────────────────────────────────────────── Контекст: S4

Требование: Расписания дежурств ДОЛЖНЫ (SHOULD) экспортироваться в
iCal/Google Calendar.

Приоритет: SHOULD Verification: Demo

### 3.3.7. SLO & Error Budget (FR-SLO)
### FR-SLO-001 Декларативное описание SLO
───────────────────────────────────────────── Контекст: S6

Требование: SLO ДОЛЖНЫ (MUST) описываться декларативно с указанием SLI,
target, rolling window. Хранятся в Git, применяются через API.
Приоритет: MUST Verification: API review

### FR-SLO-002 Непрерывное вычисление error budget
───────────────────────────────────────────── Контекст: S6

Требование: Система ДОЛЖНА (MUST) непрерывно вычислять текущий уровень
SLI, оставшийся error budget, текущий burn rate.

Приоритет: MUST Verification: Functional test против reference dataset

### FR-SLO-003 SLO dashboard
───────────────────────────────────────────── Контекст: S6, weekly
review

Требование: Система ДОЛЖНА (MUST) предоставлять dashboard со статусом
всех SLO тенанта: текущий уровень, тренд, top consumers бюджета.

Приоритет: MUST Verification: UI review

### 3.6.8. Silence & Maintenance (FR-SIL)
### FR-SIL-001 Создание silence с обязательной причиной
───────────────────────────────────────────── Контекст: S5

Требование: Silence ДОЛЖЕН (MUST) содержать: matcher, время
начала/конца, автора, причину (свободный текст + опциональная ссылка на
Jira). Rationale: «Silence без причины» = «забытый silence» =
пропущенный инцидент.

Приоритет: MUST Verification: Form validation test

### FR-SIL-002 Auto-expire silence
───────────────────────────────────────────── Контекст: S5

Требование: Silence ДОЛЖЕН (MUST) автоматически истекать по указанному
времени; максимальная длительность — 7 дней с возможностью явного
продления.

Приоритет: MUST Verification: Functional test

### FR-SIL-003 4-eyes principle для широких silence
───────────────────────────────────────────── Контекст: S5, exception E1

Требование: Silence с matcher, покрывающим > N сервисов или весь env,
ДОЛЖЕН (SHOULD) требовать подтверждения второго инженера. Rationale:
Защита от случайного «silence всего prod».

Приоритет: SHOULD Verification: Scenario test

### 3.6.9. API & Integration (FR-API)
### FR-API-001 Полнофункциональный REST/gRPC API
───────────────────────────────────────────── Контекст: общесистемное;
всё, что доступно в UI, должно быть в API Требование: Все операции,
доступные через UI, ДОЛЖНЫ (MUST) быть доступны через документированный
API. API ДОЛЖЕН (MUST) следовать принципам OpenAPI 3.0. Rationale:
Automation, CI/CD integration, GitOps, external tooling.

Приоритет: MUST Verification: API review, OpenAPI spec validation

### FR-API-002 Webhook для внешних интеграций
──────────────────────────────────────────── Контекст: S2, S3 (Jira), S5

Требование: Система ДОЛЖНА (MUST) поддерживать outbound webhooks для
событий: alert created/acked/resolved, silence created, SLO breach.

Приоритет: MUST Verification: Integration test

### FR-API-003 Стандартные интеграции
───────────────────────────────────────────── Контекст: S2, S3

Требование: Система ДОЛЖНА (SHOULD) предоставлять out-of-the-box
интеграции: Jira, PagerDuty-совместимый interface, Slack, Microsoft
Teams, ServiceNow.

Приоритет: SHOULD Verification: Per-integration test

### 3.6.10. User Interface (FR-UI)
### FR-UI-001 Unified observability UI
───────────────────────────────────────────── Контекст: S3

Требование: Метрики, логи, трейсы, алерты ДОЛЖНЫ (MUST) быть доступны в
едином UI с возможностью переключения контекста (время, сервис, tenant)
без потери выбора. Rationale: Раздельные UI = переключение вкладок =
потеря контекста = рост MTTR. Главная боль legacy stack из 3-5
инструментов.

Приоритет: MUST Verification: Usability test, E2E investigation scenario

### FR-UI-002 Дашборды как код
───────────────────────────────────────────── Контекст: S1, S6

Требование: Дашборды ДОЛЖНЫ (MUST) описываться декларативно (JSON/YAML),
версионироваться в Git, применяться через API/CI. UI-редактор ДОЛЖЕН
(SHOULD) экспортировать определение. Rationale: Dashboards-as-code =
воспроизводимость, code review, шаблонизация для onboarding.

Приоритет: MUST Verification: API review, CI demo

### FR-UI-003 Permalinks с полным контекстом
───────────────────────────────────────────── Контекст: S3 (эскалация,
post-mortem)

Требование: Любое состояние UI (запрос, временное окно, фильтры,
выбранная панель) ДОЛЖНО (MUST) быть представимо ссылкой, открываемой
другим пользователем с теми же правами. Rationale: Передача контекста
при эскалации/handoff критична для MTTR.

Приоритет: MUST Verification: Functional test

### FR-UI-004 Производительность UI под нагрузкой
───────────────────────────────────────────── Контекст: S3, on-call под
давлением; см. QAS-PERF-03

Требование: Базовые операции UI (открытие дашборда, выполнение типового
запроса) ДОЛЖНЫ (MUST) завершаться за p95 < 3 сек.

Приоритет: MUST Verification: UI performance test

### FR-UI-005 Мобильный доступ для on-call
───────────────────────────────────────────── Контекст: S2 (получение
алерта в нерабочее время)

Требование: On-call инженер ДОЛЖЕН (SHOULD) иметь mobile-friendly
интерфейс для ack алерта, просмотра контекста (top metrics, recent logs)
и инициирования эскалации.

Приоритет: SHOULD Verification: Mobile usability test

### 3.6.11. Administration & Multi-tenancy (FR-ADM)
### FR-ADM-001 Tenant lifecycle management
───────────────────────────────────────────── Контекст: S1,
multi-tenancy

Требование: Платформа ДОЛЖНА (MUST) поддерживать lifecycle тенанта:
create, configure quotas, suspend, archive, delete. Все операции — через
API и audit log.

Приоритет: MUST Verification: API test, audit log review

### FR-ADM-002 Per-tenant квоты
───────────────────────────────────────────── Контекст: общесистемное;
защита от noisy neighbor

Требование: Для каждого тенанта ДОЛЖНЫ (MUST) задаваться квоты:
ingestion rate (samples/sec, logs/sec), storage, cardinality limit,
query concurrency. Rationale: Без квот один тенант может уронить весь
кластер (cardinality explosion — самый частый incident в Prometheus).

Приоритет: MUST Verification: Load test с превышением квот

**FR-ADM-003 RBAC** ─────────────────────────────────────────────
Контекст: S5, S7, общесистемное

Требование: Система ДОЛЖНА (MUST) поддерживать role-based access control
с ролями минимум: Viewer, Editor, On-call Responder, Tenant Admin,
Platform Admin. Права назначаются на уровне tenant и (где применимо) на
уровне ресурса.

Приоритет: MUST Verification: Security audit, permission matrix test

### FR-ADM-004 SSO/OIDC integration
───────────────────────────────────────────── Контекст: enterprise
integration

Требование: Аутентификация ДОЛЖНА (MUST) поддерживать OIDC / SAML 2.0;
локальные учётки допустимы только для break-glass.

Приоритет: MUST Verification: Integration test с эталонным IdP

**FR-ADM-005 Audit log** ─────────────────────────────────────────────
Контекст: compliance, post-mortem

Требование: Все изменения конфигурации (правила, silence, RBAC,
tenants), а также действия с алертами (ack, escalate, resolve) ДОЛЖНЫ
(MUST) записываться в immutable audit log с retention >= 2 года.

Приоритет: MUST Verification: Audit log review, immutability check

### 3.3.12. Disaster Recovery & Self-monitoring (FR-DR)
### FR-DR-001 Self-monitoring через независимый канал
───────────────────────────────────────────── Контекст: S8

Требование: СМА ДОЛЖНА (MUST) мониториться независимой минимальной
инсталляцией (meta-monitoring) с отдельным notification-каналом для
критичных алертов о собственном состоянии. Rationale: Если СМА мониторит
саму себя и падает — никто не узнает. Meta-monitoring — обязательный
паттерн для observability.

Приоритет: MUST Verification: Chaos test (kill primary), check
meta-monitoring alert

### FR-DR-002 Деградация по уровням
───────────────────────────────────────────── Контекст: S8; см.
QAS-AVAIL-01

Требование: При частичном отказе система ДОЛЖНА (MUST) деградировать по
приоритетам: (1) приём метрик и алертинг сохраняются дольше всех; (2)
query/UI могут деградировать; (3) логи/трейсы могут быть отложены, но не
потеряны. Rationale: В кризис критичен alerting; UI может подождать. Без
явных приоритетов система ломается «всем одинаково».

Приоритет: MUST Verification: Chaos test scenarios

### FR-DR-003 Бэкап конфигурации
───────────────────────────────────────────── Контекст: S8

Требование: Вся конфигурация (rules, dashboards, SLO, schedules,
tenants, RBAC) ДОЛЖНА (MUST) непрерывно бэкапиться в внешнее хранилище.
RPO для конфигурации <= 5 минут.

Приоритет: MUST Verification: Restore drill

### FR-DR-004 Документированная процедура восстановления
───────────────────────────────────────────── Контекст: S8

Требование: ДОЛЖЕН (MUST) существовать runbook по полному восстановлению
СМА из бэкапа, проверенный регулярными DR-учениями (>= 1/квартал).
Rationale: Непроверенный бэкап = нет бэкапа.

Приоритет: MUST Verification: DR drill report

## 3.7. Quality Attribute Scenarios (HIGH-priority)
### 3.7.1. Performance
### QAS-PERF-01 Ingestion latency under normal load
───────────────────────────────────────────── Source: Продуктовые
сервисы (E1) через OTel collector Stimulus: Steady-state ingestion — 1M
samples/sec метрик, 500K log entries/sec, 100K spans/sec Artifact:
Ingestion pipeline СМА Environment: Production, normal operations
Response: Данные приняты, ack возвращён, данные доступны для query
Response measure: p95 ack latency < 1 сек; p99 ack latency < 3 сек;
end-to-end visibility (от send до queryable) < 30 сек Trace:
FR-ING-001..004

### QAS-PERF-02 Ingestion under burst (10x spike)
───────────────────────────────────────────── Source: Продуктовые
сервисы (E1) — incident в одном из них вызывает log storm Stimulus:
Burst до 10x от baseline в течение 5 минут Artifact: Ingestion
pipeline + storage Environment: Production, peak hours Response: (1)
backpressure активирован; (2) приоритетные данные (метрики,
P1-релевантные логи) не теряются; (3) отброшенные данные логируются с
метрикой о drop rate; (4) система не уходит в каскадный отказ Response
measure: Data loss для метрик = 0; data loss для логов <= 5%;
отсутствие impact на other tenants (см. QAS-SCAL-01); возврат к baseline
latency < 5 мин после окончания burst Trace: FR-ING-004, FR-ADM-002

### QAS-PERF-03 Query latency для типовых сценариев
───────────────────────────────────────────── Source: On-call инженер
(U2) в UI или через API Stimulus: Типовой расследовательский запрос: (a)
метрика за 1 час с агрегацией по labels; (b) поиск по логам сервиса за 1
час с filter; (c) trace search по trace_id Artifact: Query engine +
storage Environment: Production, p95 нагрузка Response: Результат
возвращён полностью или с явным указанием на partial result Response
measure: (a) метрики, 1ч окно: p95 < 2 сек, p99 < 5 сек; (b) логи, 1ч
окно: p95 < 5 сек, p99 < 15 сек; (c) trace by ID: p95 < 1 сек Trace:
FR-QRY-001..003, FR-UI-004

### 3.7.2. Availability
### QAS-AVAIL-01 Graceful degradation при отказе компонента
───────────────────────────────────────────── Source: Внутренний отказ —
теряется один из компонентов (query engine / log storage / UI backend)
Stimulus: Полный отказ одного функционального компонента Artifact:
Система в целом Environment: Production Response: Система деградирует по
приоритетам (см. FR-DR-002): - alerting продолжает работать (наивысший
приоритет); - ingestion метрик продолжает работать; - degraded функции
явно индицируются в UI; - meta-monitoring генерирует алерт о деградации
Response measure: Availability alerting pipeline >= 99.95% даже при
отказе query/UI; MTTR деградировавшего компонента <= 30 минут
(auto-recovery) или явная индикация для оператора Trace: FR-DR-001,
### FR-DR-002 QAS-AVAIL-02 Доступность при отказе зоны
───────────────────────────────────────────── Source: Облачный провайдер
Stimulus: Полный отказ одной availability zone из трёх Artifact: Вся СМА
Environment: Production, multi-AZ deployment Response: Система
продолжает функционировать без потери данных (durability через
replication >= 2 AZ); наблюдается временная деградация latency на время
rebalance Response measure: Data loss = 0; alerting downtime <= 60 сек;
full functionality recovery <= 5 минут; общая availability >= 99.9%
включая инциденты AZ Trace: FR-STO-003, FR-DR-002

### 3.7.3. Reliability (Data Durability)
### QAS-RELI-01 Durability данных после acknowledgement
───────────────────────────────────────────── Source: Любой источник
телеметрии (E1) или конфигурации Stimulus: После возврата ack клиенту:
(a) отказ одного узла storage; (b) отказ двух узлов storage; (c)
corruption на диске Artifact: Storage layer Environment: Production
Response: (a) данные доступны без задержки; (b) данные доступны после
re-replication; (c) corrupted блоки автоматически восстанавливаются из
реплик; инцидент логируется Response measure: Метрики/трейсы: durability
>= 99.99% за год; логи: durability >= 99.99% за год; алерты и audit
log: durability = 100% (synchronous replication); RPO для конфигурации
<= 5 минут Trace: FR-STO-003, FR-STO-004, FR-DR-003

### 3.7.4. Scalability
### QAS-SCAL-01 Изоляция нагрузки между тенантами (noisy neighbor)
───────────────────────────────────────────── Source: Один из тенантов
Stimulus: Tenant генерирует нагрузку в 100x от своей квоты (cardinality
explosion, log storm) Artifact: Ingestion + storage + query Environment:
Production, multi-tenant Response: (1) Tenant rate-limited по квоте; (2)
превышающие данные дропаются с алертом tenant-админу; (3) другие тенанты
не видят impact ни на latency, ни на availability Response measure:
Latency других тенантов не превышает baseline более чем на 10%; abuser
получает rate-limit response < 100ms; alert tenant-админу в течение 1
минуты Trace: FR-ADM-002, FR-ING-006

### QAS-SCAL-02 Горизонтальное масштабирование под рост
───────────────────────────────────────────── Source: Платформенная
команда (U1 SRE) — планируемый рост нагрузки в 3x за квартал Stimulus:
Постепенное добавление новых тенантов / увеличение cardinality /
увеличение log volume Artifact: Вся платформа Environment: Production
Response: Платформа масштабируется горизонтально (добавление узлов) без
изменения архитектуры и без downtime Response measure: Linear scaling до
10x текущей нагрузки (efficiency >= 70% от линейной); capacity
expansion без downtime; capacity expansion в течение 1 рабочего дня
Trace: FR-ADM-002 (квоты), tactics в Разделе 5

### 3.7.5. Security
### QAS-SEC-01 Изоляция данных между тенантами
───────────────────────────────────────────── Source: Внешний или
внутренний актор — пользователь tenant A Stimulus: Попытка
прочитать/изменить данные tenant B через API, UI или прямой query с
подменой tenant_id Artifact: Auth + authorization + storage Environment:
Production Response: Запрос отклонён с 403; попытка зафиксирована в
audit log и security alert Response measure: 100% попыток cross-tenant
access блокированы; false-positive rate авторизации < 0.01%; security
alert в течение 1 минуты Trace: FR-ING-006, FR-ADM-003, FR-ADM-005

QAS-SEC-02 Защита sensitive data в телеметрии
───────────────────────────────────────────── Source: Продуктовый сервис
— случайно логирует PII / secrets (token, password, credit card)
Stimulus: Sensitive payload в incoming log/trace Artifact: Ingestion
pipeline (PII detection/redaction) Environment: Production Response:
Sensitive поля автоматически redacted перед записью на основе
configurable patterns; detection event логируется (без самих данных) для
notification security team Response measure: Detection rate для
стандартных шаблонов (credit card, JWT, common secrets) >= 95%;
redaction latency < 10ms per log entry; zero sensitive data в persisted
storage (sample-based audit) Trace: FR-ING-002, FR-ADM-005

### 3.7.6. Observability of Observability (Self-monitoring)
### QAS-MON-01 Detection отказа самой СМА
───────────────────────────────────────────── Source: Внутренний отказ
СМА (любой компонент) Stimulus: Деградация или полный отказ компонента
СМА Artifact: Meta-monitoring (independent minimal instance)
Environment: Production Response: (1) Meta-monitoring детектирует
проблему; (2) уведомляет on-call платформенной команды через independent
канал (НЕ через primary СМА); (3) формирует context: какой компонент,
severity, impact на tenants Response measure: MTTD отказа СМА <= 2
минуты; notification доставлен через canal независимый от основной СМА;
false-positive rate meta-monitoring < 5% Trace: FR-DR-001

## 3.8. Constraints (ограничения)
### 3.8.1. Технологические constraints
|           |                                                                                                                    |                                                              |
|-----------|--------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| ID        | Constraint                                                                                                         | Обоснование                                                  |
| C-TECH-01 | Платформа развёртывается в Kubernetes (управляемый или self-hosted)                                                | Целевая среда заказчика; cloud-agnostic                      |
| C-TECH-02 | Open-source компоненты предпочтительнее proprietary при сопоставимом качестве                                      | Снижение vendor lock-in, audit-ability, community support    |
| C-TECH-03 | Обязательная поддержка OpenTelemetry как primary ingestion стандарта                                               | Стратегическое направление CNCF; будущее observability       |
| C-TECH-04 | Storage tier для горячих данных — block storage; для холодных — S3-compatible object storage                       | Cost-efficiency; стандартная практика observability платформ |
| C-TECH-05 | Все компоненты должны быть stateless или явно описывать своё state-management (избегать неявного локального state) | Cloud-native operability, scalability                        |

### 3.8.2. Регуляторные и compliance constraints
|          |                                                                                                     |                               |
|----------|-----------------------------------------------------------------------------------------------------|-------------------------------|
| ID       | Constraint                                                                                          | Обоснование                   |
| C-REG-01 | Audit log retention >= 2 года, immutable                                                           | Compliance (SOC 2, ISO 27001) |
| C-REG-02 | Поддержка data residency: возможность ограничить хранение данных конкретного тенанта одним регионом | GDPR, локальные регуляции     |
| C-REG-03 | Поддержка right-to-be-forgotten для PII в логах (best-effort, через retention/redaction)            | GDPR                          |
| C-REG-04 | Шифрование данных at-rest и in-transit                                                              | Базовое compliance требование |

### 3.8.3. Организационные constraints
|          |                                                                                           |                                                    |
|----------|-------------------------------------------------------------------------------------------|----------------------------------------------------|
| ID       | Constraint                                                                                | Обоснование                                        |
| C-ORG-01 | Платформенная команда (SRE) — ~5-7 человек на старте, не более 15 на горизонте 2 года     | Влияет на operational complexity платформы         |
| C-ORG-02 | Продуктовые команды (E1) — self-service, без участия SRE в день-к-день операциях          | Архитектура должна поддерживать self-service model |
| C-ORG-03 | On-call ротация — 1 неделя primary + 1 неделя secondary, 6-8 человек в ротации на команду | Влияет на UX и расписания (FR-OCM)                 |
| C-ORG-04 | Все изменения конфигурации должны быть auditable (кто/когда/что)                          | SOC 2, ISO 27001, post-mortem analysis             |

### 3.8.4. Cost constraints
|           |                                                                                                                  |                                          |
|-----------|------------------------------------------------------------------------------------------------------------------|------------------------------------------|
| ID        | Constraint                                                                                                       | Обоснование                              |
| C-COST-01 | Общая стоимость инфраструктуры СМА не должна превышать ~3-5% от стоимости infrastructure наблюдаемых сервисов    | Стандартный benchmark observability cost |
| C-COST-02 | Cost per ingested GB логов должен быть в пределах рыночного benchmark managed решений (Datadog/Splunk) минус 50% | Бизнес-case проекта (build vs buy)       |
| C-COST-03 | Холодное хранение должно использовать object storage tier (Glacier-like) для данных старше 30 дней               | Cost-efficiency для long retention       |

### 3.8.5. Tactics для MEDIUM-priority QA
Архитектурные тактики, которые будут детализированы в Разделе 5:

### Usability (см. также FR-UI):
- Unified UI с консистентной навигацией между signals

- Permalinks и saved queries для передачи контекста

- Mobile-friendly views для on-call

- Onboarding wizard для новых тенантов и команд

### Maintainability:
- Rules/dashboards/SLO as code (Git-based)

- Декларативная конфигурация всех ресурсов

- Стандартизованные deployment patterns (Helm/operators)

- Comprehensive self-monitoring (см. QAS-MON-01)

### Interoperability:
- OpenTelemetry как primary ingestion path

- PromQL для метрик, LogQL-совместимость для логов

- OpenAPI 3.0 для всех external API

- Webhook-based outbound integration

### Cost-efficiency:
- Tiered storage (FR-STO-002)

- Per-tenant квоты (FR-ADM-002) для cost control

- Configurable retention per data type (FR-STO-001)

- Sampling tactics для трейсов (head/tail sampling)

## 3.9. Матрица трассировки
Матрица показывает, какие FR/QAS происходят из каких сценариев CONOPS, и
какие компоненты будут их реализовывать (forward-reference в Раздел 5).

### 3.9.1. CONOPS → Requirements
|                              |                                                                      |                                        |
|------------------------------|----------------------------------------------------------------------|----------------------------------------|
| Scenario CONOPS              | Связанные FR                                                         | Связанные QAS                          |
| S1 Onboarding нового сервиса | FR-ING-005, FR-ADM-001, FR-ADM-002, FR-RUL-001, FR-UI-002            | —                                      |
| S2 Срабатывание алерта       | FR-RUL-001..005, FR-NTF-001..005, FR-UI-005                          | QAS-PERF-01, QAS-AVAIL-01              |
| S3 Расследование инцидента   | FR-QRY-001..004, FR-UI-001..003, FR-API-002                          | QAS-PERF-03, QAS-AVAIL-01              |
| S4 Ротация on-call           | FR-OCM-001..003                                                      | —                                      |
| S5 Silence на maintenance    | FR-SIL-001..003, FR-ADM-005                                          | —                                      |
| S6 SLO weekly review         | FR-SLO-001..003, FR-RUL-003                                          | —                                      |
| S7 Капасити-планирование     | FR-QRY-001, FR-ADM-002                                               | QAS-SCAL-01, QAS-SCAL-02               |
| S8 DR / отказ СМА            | FR-DR-001..004                                                       | QAS-AVAIL-01, QAS-AVAIL-02, QAS-MON-01 |
| Общесистемные                | FR-ING-004, FR-ING-006, FR-STO-001..004, FR-API-001, FR-ADM-001..005 | QAS-RELI-01, QAS-SEC-01..02            |

### 3.9.2. Requirements → Components (forward-reference)
|           |                                                               |
|-----------|---------------------------------------------------------------|
| Group FR  | Будущие компоненты (Раздел 5)                                 |
| FR-ING-\* | Ingestion Gateway, OTel Collector, Tenant Router              |
| FR-STO-\* | Metrics Store, Log Store, Trace Store, Object Storage Adapter |
| FR-QRY-\* | Query Engine, Correlation Service                             |
| FR-RUL-\* | Rule Engine, Evaluator                                        |
| FR-NTF-\* | Alertmanager, Notification Router, Channel Adapters           |
| FR-OCM-\* | On-call Schedule Service                                      |
| FR-SLO-\* | SLO Engine                                                    |
| FR-SIL-\* | Silence Service (часть Alertmanager)                          |
| FR-API-\* | API Gateway, GraphQL/REST facades                             |
| FR-UI-\*  | Web UI, Mobile UI, Dashboard Service                          |
| FR-ADM-\* | Tenant Service, Auth Service, RBAC Engine, Audit Service      |
| FR-DR-\*  | Meta-monitoring Stack, Backup Service                         |
