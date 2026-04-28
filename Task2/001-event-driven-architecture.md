# ADR-001. Event-Driven архитектура GoFuture

- **Статус:** Accepted
- **Дата:** 2026-04-28
- **Контекст:** Задание 2
- **Автор:** Вячеслав Стеблецов

## Контекст

GoFuture должна обрабатывать до 500 000 конкурентных поездок, поддерживать динамическое ценообразование в реальном времени и масштабироваться по регионам.

Для этого используется Event-Driven Architecture на базе Kafka: доменные сервисы публикуют события, потоковые обработчики считают агрегаты спроса/предложения, а мониторинг контролирует задержки, ошибки и доступность событийной платформы.

## Требования

| **№** | **Требование** |
| :-: | :- |
| 1 | Использовать доменные события (`BookingCreated`, `DriverLocationUpdated`, `PaymentCaptured` и др.) |
| 2 | Определить Kafka topics с партиционированием по регионам |
| 3 | Использовать Kafka Streams / Flink для real-time обработки |
| 4 | Выбрать вид Saga для процесса поездки |
| 5 | Обеспечить надёжную доставку событий |
| 6 | Определить мониторинг, метрики и инструменты наблюдаемости |
| 7 | Отразить инструменты мониторинга на C2-диаграмме |

---

## Решение

### 1. Доменные события

| Домен | Основные события |
| :- | :- |
| Booking | `BookingCreated`, `BookingCancelled`, `RideStarted`, `RideCompleted` |
| Driver | `DriverLocationUpdated`, `DriverAvailable`, `DriverAssigned`, `DriverStatusChanged` |
| Pricing | `PriceCalculated`, `SurgeMultiplierChanged`, `ZoneDemandChanged` |
| Payments | `PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed`, `PayoutCreated` |
| Fraud | `FraudCheckRequested`, `FraudCheckPassed`, `FraudCheckFailed` |
| Notification | `NotificationRequested`, `NotificationSent`, `NotificationFailed` |

---

### 2. Kafka topics и партиционирование

Основной принцип партиционирования — `region_id + aggregate_id`.

Это позволяет:

- масштабировать обработку независимо по регионам;
- сохранять порядок событий внутри агрегата;
- поддерживать data residency и локальную обработку.

| Topic | Partition key | Основные события |
| :- | :- | :- |
| `booking.events.v1` | `region_id + booking_id` | `BookingCreated`, `BookingCancelled`, `RideStarted`, `RideCompleted` |
| `driver.events.v1` | `region_id + driver_id` | `DriverLocationUpdated`, `DriverAvailable`, `DriverStatusChanged` |
| `pricing.events.v1` | `region_id + zone_id` | `PriceCalculated`, `SurgeMultiplierChanged`, `ZoneDemandChanged` |
| `payment.events.v1` | `region_id + payment_id` | `PaymentAuthorized`, `PaymentCaptured`, `PaymentFailed` |
| `fraud.events.v1` | `region_id + booking_id` | `FraudCheckRequested`, `FraudCheckPassed`, `FraudCheckFailed` |
| `notification.events.v1` | `region_id + user_id` | `NotificationRequested`, `NotificationSent`, `NotificationFailed` |
| `*.dlq.v1` | original event key | Ошибочные события |

---

### 3. Stream processing

Для real-time обработки используется **Kafka Streams / Flink**.

| Сценарий | Входные события | Результат |
| :- | :- | :- |
| Динамическое ценообразование | `BookingCreated`, `DriverLocationUpdated`, `RideCompleted` | Агрегаты спроса/предложения по зонам |
| Оптимизация водителей | `DriverLocationUpdated`, `DriverAvailable`, `BookingCreated` | Рекомендации по перераспределению водителей |
| Антифрод | `BookingCreated`, `PaymentAuthorized`, `DriverLocationUpdated` | Fraud signals |
| Real-time аналитика | Все доменные события | Витрины в ClickHouse |

---

### 4. Saga

Для процесса поездки используется **оркестрируемая Saga**.

Причина выбора:

- Booking Service является владельцем жизненного цикла поездки;
- процесс бизнес-критичный;
- требуется контролировать порядок шагов;
- нужны компенсирующие действия.

Основные события Saga (Ride Booking):

- `BookingCreated`
- `FraudCheckRequested` -> `FraudCheckPassed` / `FraudCheckFailed`
- `PriceCalculated`
- `DriverAssigned` / `DriverAssignmentFailed`
- `PaymentAuthorized` / `PaymentFailed`
- `RideStarted`
- `RideCompleted`
- `PaymentCaptured`
- `BookingCancelled`
- `NotificationRequested`

Happy path:

`BookingCreated` -> `FraudCheckPassed` -> `PriceCalculated` -> `DriverAssigned` -> `PaymentAuthorized` -> `RideStarted` -> `RideCompleted` -> `PaymentCaptured` -> `NotificationRequested`

---

### 5. Надёжная доставка

| Механизм | Назначение |
| :- | :- |
| Transactional Outbox + CDC | Запись бизнес-данных и события в одной транзакции; Debezium публикует событие в Kafka |
| Idempotency | Потребители хранят `event_id` и не применяют событие повторно |
| Retry | Временные ошибки обрабатываются повторной доставкой |
| DLQ | Ошибочные события после лимита retry попадают в `*.dlq.v1` |
| Ordering | Порядок гарантируется внутри партиции по `region_id + aggregate_id` |
| Schema Registry | Версионирование событий и backward-compatible evolution |

---

## Мониторинг

### Подход

Мониторинг строится вокруг SLI/SLO событийной платформы:

- задержка публикации и обработки событий;
- consumer lag;
- ошибки producers/consumers;
- состояние Kafka;
- состояние CDC;
- успешность Saga;
- задержка обновления динамической цены.

Цель — быстро обнаруживать деградацию Kafka, остановку CDC, рост DLQ, сбои Saga и задержки real-time pricing.

### Метрики

| Группа | Метрики |
| :- | :- |
| Kafka | `kafka_broker_up`, `kafka_consumer_lag`, `kafka_under_replicated_partitions`, `kafka_offline_partitions_count` |
| Producer | `producer_error_rate`, `producer_retry_rate`, `producer_record_send_latency_ms` |
| Consumer | `consumer_lag`, `consumer_processing_time_ms`, `consumer_error_rate`, `consumer_dlq_events_total` |
| Stream processing | `stream_processing_latency_ms`, `stream_lag`, `pricing_update_latency_ms` |
| Outbox / CDC | `debezium_connector_status`, `debezium_lag_ms`, `outbox_unpublished_events` |
| Saga | `saga_completed_total`, `saga_failed_total`, `saga_compensated_total`, `saga_duration_ms` |
| Business | `active_rides_total`, `booking_creation_rate`, `driver_location_update_rate`, `payment_success_rate` |

### Инструменты

| Инструмент | Назначение |
| :- | :- |
| Prometheus | Сбор технических и бизнес-метрик |
| Grafana | Дашборды Kafka, Saga, Pricing, CDC |
| Loki | Централизованные логи |
| Alertmanager | Алерты по SLO/SLA |
| OpenTelemetry | Распределённая трассировка запросов и событий |
| Kafka Exporter | Метрики Kafka и consumer lag |
| Debezium metrics | Контроль CDC lag и состояния connector’ов |
| Flink / Kafka Streams metrics | Метрики потоковой обработки |

---

## Последствия

### Плюсы

- Система поддерживает real-time обработку событий.
- Kafka сглаживает пиковые нагрузки.
- Динамическое ценообразование получает актуальные данные.
- Домены масштабируются независимо.
- Transactional Outbox + CDC снижает риск потери событий.
- Saga управляет распределённым процессом поездки.

### Минусы / риски

- Растёт сложность эксплуатации.
- Требуется зрелый мониторинг Kafka и CDC.
- Потребители должны быть идемпотентными.
- Необходимо управлять версиями событий.
- Eventual consistency становится частью архитектуры.
