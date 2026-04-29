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

Используется **Global Edge Platform**:

- GeoDNS + latency routing;
- health checks;
- автоматическое переключение региона;
- Anycast сеть.

---

### 3. Репликация

#### PostgreSQL
- async logical replication;
- primary -> replica;
- eventual consistency.

#### Kafka
- MirrorMaker 2;
- репликация топиков между регионами.

---

### 4. Failover стратегия

- health checks на уровне edge;
- автоматическое исключение недоступного региона;
- переключение трафика на ближайший доступный регион;
- promotion replica DB -> primary.

---

### 5. Сценарии отказа

| Сценарий | Реакция системы |
| :- | :- |
| Регион недоступен | Geo-routing переключает трафик |
| API недоступен | traffic reroute |
| DB отказ | replica promotion |
| Kafka недоступен | fallback на другой регион |

---

### 6. Безопасность

Edge layer включает:

- DDoS protection;
- WAF;
- rate limiting;
- TLS termination.

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