# ADR-001. Мультитенантная платформа, IAM и onboarding партнёров

- **Статус:** Accepted
- **Дата:** 2026-04-29
- **Контекст:** Задание 5
- **Автор:** Вячеслав Стеблецов

---

## Контекст

GoFuture запускается в новых регионах через крупных партнёров-перевозчиков.

Характеристики tenant:

- десятки партнёров;
- десятки тысяч водителей на партнёра;
- сотни тысяч поездок;
- строгие требования к изоляции данных и compliance.

Требуется:

- безопасная мультитенантная архитектура;
- централизованный IAM с SSO;
- быстрый onboarding новых партнёров;
- мониторинг на уровне tenant.

---

## Требования

| **№** | **Требование** |
| :-: | :- |
| 1 | Полная изоляция данных между партнёрами |
| 2 | IAM с SSO и RBAC |
| 3 | Автоматизированный onboarding |
| 4 | Tenant-level мониторинг |
| 5 | Audit действий пользователей и сервисов |
| 6 | Возможность кастомизации |

---

## Решение

### 1. Модель мультитенантности

Выбрана модель **Database-per-Tenant**.

Каждый tenant получает:

- отдельную базу данных;
- отдельные credentials;
- отдельные backup / restore;
- отдельные resource limits;
- изоляцию на уровне хранения данных.

Дополнительно используется `tenant_id`:

- в JWT;
- в Kafka событиях;
- в логах и метриках;
- в audit.

Назначение `tenant_id` — трассировка, а не изоляция.

---

### 2. IAM

IAM является shared platform-компонентом, но изоляция обеспечивается через:

- отдельные realm / organization / tenant namespace для каждого партнёра;
- tenant-scoped roles и permissions;
- запрет cross-tenant access на уровне policies;
- `tenant_id` claim в access token;
- audit log всех действий;
- отдельные service accounts для интеграций каждого tenant.

JWT содержит:

- `tenant_id`
- `roles`
- `permissions`
- `region`

---

### 3. Onboarding партнёра

Процесс автоматизирован:

1. Создание tenant (Tenant Registry)
2. Provision DB (Database-per-Tenant)
3. Создание IAM realm / roles
4. Создание Kafka topics + ACL
5. Настройка feature flags
6. Настройка monitoring dashboards
7. Генерация API credentials
8. Smoke tests
9. Активация tenant

---

### 4. Кастомизация

Реализуется через:

- feature flags;
- tenant configuration;
- pricing rules;
- региональные интеграции.

Без форка кодовой базы.

---

### 5. Мониторинг

Мониторинг tenant-aware:

| Уровень | Метрики                               |
| :- |:--------------------------------------|
| Platform | availability, latency                 |
| Tenant | rides, payments, error rate, latency p95/p99 |
| IAM | login failures                        |
| Kafka | consumer lag per tenant               |
| DB | connections, load per tenant          |
| Onboarding | provisioning time                     |

Дополнительно контролируются:

- DLQ events per tenant;
- audit anomalies;
- cross-tenant access violations (security critical);

Мониторинг строится на основе SLI/SLO с алертингом через Alertmanager.

Инструменты:

- Prometheus
- Grafana
- Loki
- Alertmanager
- OpenTelemetry

---

## Таблица ролей

| Роль | Доступ | Возможности |
| :- | :- | :- |
| Platform Admin | Все tenant | Управление платформой |
| Partner Admin | Свой tenant | Управление пользователями |
| Operator | Свой tenant | Операционные задачи |
| Finance | Финансы tenant | Платежи и отчёты |
| Support | Ограниченный | Поддержка |
| Analyst | Обезличенные данные | BI |
| API Client | Ограниченный | Интеграции |
| Auditor | Read-only | Аудит |

---

## Альтернативы

### Shared DB + RLS
❌ Риск утечки данных

### Schema-per-Tenant
⚠️ Общий кластер, слабее изоляция

### Database-per-Tenant
✅ Выбрано

### Single deployment per tenant
❌ Слишком дорого

---

## Выбор

Database-per-Tenant выбран, потому что:

- небольшое количество tenant (десятки);
- высокий уровень требований к безопасности;
- упрощает compliance;
- снижает риск cross-tenant утечек.

---

## Последствия

### Плюсы

- максимальная изоляция;
- проще аудит и compliance;
- независимые backup/restore;
- меньше рисков безопасности.

### Минусы

- выше операционная сложность;
- больше инфраструктурных ресурсов;
- требуется automation provisioning.

---

## Компромиссы

- увеличена сложность ради безопасности;
- централизованный IAM остаётся shared, но доступы, роли и политики строго изолированы по `tenant_id`