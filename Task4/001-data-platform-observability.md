# ADR-001. Data Platform для ML, BI и контроля качества данных

- **Статус:** Accepted
- **Дата:** 2026-04-29
- **Контекст:** Задание 4
- **Автор:** Вячеслав Стеблецов

---

## Контекст

GoFuture требуется единая платформа сбора, обработки и анализа данных из микросервисов.

Платформа должна поддерживать:

- ML-модели динамического ценообразования;
- прогнозирование спроса;
- обнаружение мошенничества;
- BI-аналитику для бизнеса;
- контроль качества данных.

---

## Требования

| **№** | **Требование** |
| :-: | :- |
| 1 | Собирать события из всех доменных микросервисов |
| 2 | Поддерживать near real-time обработку данных |
| 3 | Интегрироваться с ML-платформой |
| 4 | Интегрироваться с BI-инструментами |
| 5 | Хранить исторические данные для аналитики и обучения моделей |
| 6 | Обеспечивать контроль качества данных |

---

## Решение

### 1. Сбор данных

Источниками данных являются доменные сервисы:

- Booking Service;
- Driver Service;
- Pricing Service;
- Payments Service;
- Fraud Service;
- Geography Service;
- Notification Service.

Сервисы публикуют доменные события в Kafka через Transactional Outbox + CDC.

Основные события:

| Домен | Примеры событий |
| :- | :- |
| Booking | `BookingCreated`, `RideStarted`, `RideCompleted`, `BookingCancelled` |
| Driver | `DriverLocationUpdated`, `DriverAvailable`, `DriverAssigned` |
| Pricing | `PriceCalculated`, `SurgeMultiplierChanged` |
| Payments | `PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed` |
| Fraud | `FraudCheckPassed`, `FraudCheckFailed` |
| Geography | `RouteCalculated`, `ZoneDemandChanged` |
| Notification | `NotificationSent`, `NotificationFailed` |
---

### 2. Data pipeline

Используется Lambda/Kappa-style pipeline:

| Слой | Назначение |
| :- | :- |
| Kafka | Сбор доменных событий |
| Stream Processing | Near real-time агрегации и признаки |
| Data Lake | Хранение сырых исторических данных |
| ClickHouse | Аналитические витрины и BI |
| Feature Store | Online/offline признаки для ML |
| ML Platform | Обучение и serving моделей |
| BI Tool | Дашборды и отчёты |

Поток данных:

Microservices -> Kafka -> Stream Processing -> Data Lake / ClickHouse / Feature Store -> ML Platform / BI

---

### 3. ML-платформа

ML-платформа использует данные для:

| Use case | Данные |
| :- | :- |
| Dynamic pricing | спрос, предложение, зона, время, погода, история поездок |
| Demand forecasting | исторические поездки, геозоны, временные паттерны |
| Fraud detection | поездки, платежи, устройства, частота отмен |

Модели получают признаки через Feature Store:

- offline features — для обучения;
- online features — для inference в production.

---

### 4. BI-интеграция

Для BI используется ClickHouse + DataLens.

ClickHouse хранит:

- агрегаты поездок;
- финансовые метрики;
- SLA/SLO метрики;
- показатели спроса и предложения;
- fraud-метрики.

DataLens используется для:

- операционных дашбордов;
- продуктовой аналитики;
- финансовой отчётности;
- мониторинга регионов.

---

## Data Quality

### 1. Контроль качества данных

| Механизм | Назначение |
| :- | :- |
| Schema Registry | Контроль схем событий и совместимости |
| Data Contracts | Явные контракты между producer и consumer |
| Validation Rules | Проверка обязательных полей и типов |
| Deduplication | Удаление дублей по `event_id` |
| Freshness checks | Контроль актуальности данных |
| Completeness checks | Контроль полноты данных |
| Reconciliation | Сверка данных между source-сервисом и витриной |
| DLQ | Изоляция некорректных событий |

---

### 2. Минимальные проверки

| Проверка | Пример |
| :- | :- |
| Schema validation | событие соответствует Avro/Protobuf schema |
| Required fields | `event_id`, `event_time`, `region_id`, `aggregate_id` обязательны |
| Value range | цена поездки > 0, координаты в допустимом диапазоне |
| Freshness | события по поездкам приходят с задержкой < 1 минуты |
| Uniqueness | `event_id` не должен обрабатываться повторно |
| Referential consistency | `payment_id` должен ссылаться на существующий `booking_id` |

---

### 3. Мониторинг качества данных

| Метрика | Назначение |
| :- | :- |
| `data_freshness_lag_ms` | задержка поступления данных |
| `invalid_events_total` | количество невалидных событий |
| `duplicate_events_total` | количество дублей |
| `schema_validation_failed_total` | ошибки проверки схемы |
| `dlq_events_total` | события в DLQ |
| `reconciliation_mismatch_total` | расхождения между source и витриной |
| `feature_missing_rate` | доля отсутствующих признаков |

---

## Альтернативы

### 1. Только ClickHouse без Data Lake

❌ Не подходит: ClickHouse удобен для аналитики, но не заменяет долговременное хранение сырых данных для ML.

### 2. Только batch ETL

❌ Не подходит: динамическое ценообразование и fraud detection требуют near real-time данных.

### 3. Прямые подключения BI к сервисным БД

❌ Не подходит: создаёт нагрузку на production-БД и нарушает границы доменов.

### 4. Отдельные пайплайны для каждого домена

⚠️ Возможны, но усложняют governance и контроль качества.

---

## Выбор

Выбрана единая event-driven data platform:

- Kafka как событийный backbone;
- Stream Processing для near real-time обработки;
- Data Lake для исторических сырых данных;
- ClickHouse для аналитики;
- Feature Store для ML-признаков;
- DataLens для BI.

Такой подход обеспечивает масштабируемость, независимость доменов и единый контроль качества данных.

---

## Последствия

### Плюсы

- Единый поток данных из всех микросервисов.
- Поддержка ML и BI без нагрузки на production-БД.
- Near real-time данные для pricing и fraud detection.
- Возможность переобучения моделей на исторических данных.
- Контроль качества данных на уровне схем, контрактов и метрик.

### Минусы

- Усложняется data governance.
- Требуется поддержка Schema Registry и Data Contracts.
- Возможны задержки в stream processing.
- Нужно мониторить качество данных и DLQ.

---

## Компромиссы

- Принимается eventual consistency между сервисами и аналитикой.
- Увеличивается инфраструктурная сложность ради ML/BI use cases.
- Сырые данные хранятся отдельно от аналитических витрин.