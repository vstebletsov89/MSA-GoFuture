# ADR-001. Global Deployment, Geo-routing и Failover стратегия

- **Статус:** Accepted
- **Дата:** 2026-04-29
- **Контекст:** Задание 3
- **Автор:** Вячеслав Стеблецов

---

## Контекст

GoFuture выходит на рынки Юго-Восточной Азии и Южной Америки.  
Требуется обеспечить:

- 99.99% availability;
- низкую latency (<150 ms);
- устойчивость к отказам регионов;
- соответствие локальным требованиям.

---

## Требования

| **№** | **Требование** |
| :-: | :- |
| 1 | Multi-region deployment (≥ 3 региона) |
| 2 | Геомаршрутизация пользователей |
| 3 | Репликация данных |
| 4 | Failover без downtime |
| 5 | Edge-защита (DDoS, WAF) |
| 6 | Соответствие локальным регуляциям |

---

## Решение

### 1. Регионы

| Регион | Роль | Обоснование |
| :- | :- | :- |
| Singapore (SEA-1) | Primary SEA | Низкая latency, развитая инфраструктура |
| Jakarta (SEA-2) | DR SEA | Географическая изоляция |
| São Paulo (SA-1) | Primary SA | Основной рынок LATAM |
| Santiago (SA-2) | DR SA | Failover для региона |

---

### 2. Geo-routing

Используется **Global Edge Platform** (Cloudflare / Global Traffic Manager):

- GeoDNS + latency-based routing;
- Anycast сеть;
- health checks региональных API Gateway;
- автоматическое исключение деградировавшего региона;
- WAF, DDoS protection, rate limiting.

Поток трафика:

User -> Global Edge Platform -> WAF / DDoS protection -> Regional Firewall -> ближайший healthy API Gateway

---

### 3. Репликация

| Данные | Модель репликации | Обоснование |
| :- | :- | :- |
| Booking | Regional primary + async replication | Поездка имеет одного владельца записи в домашнем регионе |
| Payments | Regional primary + async replication + audit log | Платежи требуют идемпотентности и трассируемости |
| Driver location | Event-driven replication | Высокочастотные события, важна низкая задержка |
| Pricing | Локальные stream aggregates | Цена считается на основе локального спроса и предложения |
| Analytics | Async event replication | Аналитика допускает задержку |
| Reference data | Multi-region replication | Справочники нужны во всех регионах |

Механизмы:

- PostgreSQL async logical replication;
- Kafka MirrorMaker 2 / Cluster Linking;
- Transactional Outbox + CDC для надёжной публикации событий.

---

### 4. Failover стратегия

- health checks на уровне edge;
- автоматическое исключение недоступного региона;
- переключение трафика на ближайший доступный регион;
- promotion replica DB -> primary.

---

### 5. Сценарии отказа

| Сценарий | Детекция | Реакция системы |
| :- | :- | :- |
| Регион недоступен | health checks / synthetic monitoring | Geo-routing переключает трафик в DR-регион |
| API Gateway недоступен | 5xx / latency p95-p99 | регион исключается из маршрутизации |
| PostgreSQL primary недоступен | DB health check / replication lag | promotion replica -> primary |
| Kafka недоступен | broker health / consumer lag | переключение на региональную реплику Kafka |
| Stream processing деградировал | processing lag | restart job + replay событий |
| Payment provider недоступен | payment error rate | circuit breaker + fallback provider |
| WAF / Edge деградировал | edge health checks | переключение на резервный edge / fallback routing |

---

### 6. Безопасность

Edge и сетевой слой включают:

- DDoS protection;
- WAF;
- rate limiting;
- TLS termination;
- Regional Firewall / Security Groups;
- private networking между сервисами.

---

### 7. Compliance

- данные пользователей обрабатываются в пределах региона;
- PII не реплицируется глобально;
- соответствие локальным законам (data residency);
- шифрование in transit / at rest.

---

## Альтернативы

### 1. Single-region
❌ Не соответствует SLA

### 2. Active-passive
⚠️ Проще, но хуже latency

### 3. Multi-master DB
❌ Высокая сложность и конфликтность

### 4. Global Kafka cluster
❌ Высокая latency и сложность

---

## Выбор

Выбран **multi-region active-active + async replication**, потому что:

- обеспечивает низкую latency;
- масштабируется горизонтально;
- выдерживает отказ региона;
- проще операционно, чем multi-master.

---

## Последствия

### Плюсы

- высокая доступность (99.99%);
- низкая latency;
- устойчивость к отказам;
- масштабируемость.

### Минусы

- eventual consistency;
- сложность инфраструктуры;
- необходимость мониторинга replication lag.

---

## Компромиссы

- отказ от strong consistency;
- увеличение сложности ради SLA.