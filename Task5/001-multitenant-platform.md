# ADR-001. Мультитенантная платформа, IAM и onboarding партнёров

- **Статус:** Accepted
- **Дата:** 2026-04-29
- **Контекст:** Задание 5
- **Автор:** Вячеслав Стеблецов

---

## Контекст

GoFuture должна быстро подключать партнёров в новых регионах с полной изоляцией данных и возможностью кастомизации функциональности.

Партнёрами могут быть:

- локальные сервисы такси;
- банки;
- e-commerce партнёры;
- корпоративные клиенты;
- региональные операторы.

Требуется обеспечить безопасную мультитенантную модель, централизованный IAM, автоматизированный onboarding и мониторинг по каждому tenant.

---

## Требования

| **№** | **Требование** |
| :-: | :- |
| 1 | Изоляция данных между партнёрами |
| 2 | IAM с SSO и RBAC |
| 3 | Автоматизированный onboarding партнёров |
| 4 | Возможность кастомизации функциональности |
| 5 | Мониторинг tenant-level метрик |
| 6 | Аудит действий пользователей и сервисов |

---

## Решение

### 1. Модель мультитенантности

Выбрана гибридная модель:

| Уровень | Модель |
| :- | :- |
| Application | `tenant_id` в каждом запросе и событии |
| API | tenant-aware routing через API Gateway |
| Data | отдельная схема / БД для крупных tenant, row-level isolation для малых tenant |
| Events | `tenant_id` в key/header каждого события |
| Observability | метрики, логи и алерты размечаются `tenant_id` |

Для стратегических партнёров используется **database/schema per tenant**.  
Для малых партнёров допускается **shared database + row-level security**.

---

### 2. IAM

Используется централизованная IAM-система:

- SSO через OIDC / SAML;
- RBAC для ролей;
- tenant-scoped permissions;
- service accounts для интеграций;
- audit log для действий пользователей и сервисов.

JWT / access token содержит:

- `tenant_id`;
- `user_id`;
- `roles`;
- `permissions`;
- `region`;
- `partner_type`.

---

### 3. Onboarding партнёра

Onboarding выполняется через Partner Onboarding Service.

Процесс:

1. Создание tenant record.
2. Назначение региона и модели изоляции данных.
3. Создание IAM realm / groups / roles.
4. Создание схемы БД или настройка RLS.
5. Создание Kafka topics / ACL.
6. Создание feature flags и конфигурации партнёра.
7. Настройка dashboards и alerts.
8. Выпуск API credentials / service accounts.
9. Smoke tests.
10. Активация tenant.

---

### 4. Кастомизация

Кастомизация выполняется через:

- feature flags;
- tenant-specific configuration;
- pricing rules per tenant;
- branding и настройки корпоративного портала;
- региональные payment/map integrations.

Кодовая база сервисов остаётся общей, различия выносятся в конфигурацию.

---

### 5. Мониторинг

Мониторинг выполняется на уровне платформы и tenant.

| Уровень | Метрики |
| :- | :- |
| Platform | availability, latency, error rate, saturation |
| Tenant | requests, active rides, payment success rate, SLA |
| IAM | login failures, token errors, permission denied |
| Onboarding | provisioning duration, failed steps |
| Data | data isolation violations, replication lag |
| Kafka | consumer lag by tenant, DLQ events |

Инструменты:

- Prometheus;
- Grafana;
- Loki;
- Alertmanager;
- OpenTelemetry;
- Audit Log Store.

---

## Таблица ролей и доступов

| Роль | Доступ к данным | Возможности |
| :- | :- | :- |
| Platform Admin | Все tenant, все регионы | Управление платформой, настройками, инцидентами |
| Partner Admin | Только свой tenant | Управление пользователями, настройками, интеграциями |
| Partner Operator | Только свой tenant | Просмотр поездок, водителей, операций |
| Finance Manager | Финансовые данные своего tenant | Платежи, выплаты, отчёты |
| Support Agent | Ограниченный доступ своего tenant | Поиск поездок, помощь пользователям |
| Data Analyst | Обезличенные данные своего tenant | BI-отчёты и аналитика |
| Developer / API Client | Только разрешённые API | Интеграции через service account |
| Auditor | Read-only audit logs | Проверка действий и соответствия требованиям |

---

## Альтернативы

### 1. Single-tenant deployment per partner

❌ Максимальная изоляция, но слишком высокая стоимость и долгий onboarding.

### 2. Shared DB без tenant isolation

❌ Дешевле, но высокий риск утечки данных между партнёрами.

### 3. Только row-level security для всех tenant

⚠️ Подходит для малых партнёров, но слабее для крупных стратегических клиентов.

### 4. Database per tenant для всех

⚠️ Сильная изоляция, но высокая операционная сложность при большом количестве tenant.

---

## Выбор

Выбрана **гибридная multi-tenant модель**:

- крупные партнёры: schema/database per tenant;
- малые партнёры: shared database + row-level security;
- все сервисы tenant-aware;
- IAM централизован;
- onboarding автоматизирован.

Такой подход балансирует безопасность, стоимость и скорость запуска новых партнёров.

---

## Последствия

### Плюсы

- Быстрый запуск партнёров в новых регионах.
- Изоляция данных между tenant.
- Централизованное управление доступом.
- Единый мониторинг tenant-level SLA.
- Возможность кастомизации без форка кодовой базы.

### Минусы / риски

- Усложняется data governance.
- Требуется строгий контроль `tenant_id` во всех сервисах и событиях.
- Нужно тестировать изоляцию данных.
- Возрастает сложность IAM и onboarding pipeline.

---

## Компромиссы

- Принимается гибридная модель изоляции вместо одного универсального подхода.
- Увеличивается сложность платформы ради скорости подключения партнёров.
- Feature flags и tenant configuration требуют строгого governance.