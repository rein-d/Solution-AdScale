# Task 3 — данные, масштабирование, отказоустойчивость

## Состав

| Файл | Содержание | Роль |
|---|---|---|
| [database-strategy.md](database-strategy.md) | Выбор БД по сервисам (Database per Service), владение данными, retention, бэкапы, RPO/RTO, 12-factor | Пункты 1, 5 — хранилища и надёжность |
| [scaling.md](scaling.md) | Master–slave репликация, шардирование и ключи, read replicas, CQRS, пороги/триггеры масштабирования, мультирегион | Пункт 2 — масштабирование |
| [caching.md](caching.md) | Что кешируем и чего нет, кластеры Redis, TTL, инвалидация, прогрев, поведение при сбоях | Пункт 3 — кеширование |
| [event-streaming.md](event-streaming.md) | Топики Kafka, схемы (Avro), consumer groups, retention, outbox/DLQ/replay, мультирегион | Пункт 4 — потоковая обработка |
| [diagrams/](diagrams/) | PlantUML-схемы по топологии, развёртыванию, владению данными, шардированию, кешу, пайплайну событий и failover | Схемы |

## Диаграммы

| Файл | Тип | Что показывает |
|---|---|---|
| [diagrams/C4_Container_DataTopology.puml](diagrams/C4_Container_DataTopology.puml) | C4 container | Сервисы и их хранилища, кто куда пишет/читает |
| [diagrams/Data_Ownership_Map.puml](diagrams/Data_Ownership_Map.puml) | C4 container | Владение наборами данных и допустимые способы чтения |
| [diagrams/C4_Deployment_DataTopology.puml](diagrams/C4_Deployment_DataTopology.puml) | C4 deployment | 3 AZ, sync/async репликация, DR-регион, RPO/RTO |
| [diagrams/Scaling_Sharding_Map.puml](diagrams/Scaling_Sharding_Map.puml) | C4 container | Репликация vs шардирование vs партиционирование, ключи |
| [diagrams/Caching_Layers.puml](diagrams/Caching_Layers.puml) | C4 container | Слои L0–L3, TTL/инвалидация, раздельные Redis-кластеры |
| [diagrams/Sequence_EventPipeline.puml](diagrams/Sequence_EventPipeline.puml) | sequence | Beacon → Kafka → Statistics/Billing/Reporting, дедуп и DLQ |
| [diagrams/Data_DR_Failover.puml](diagrams/Data_DR_Failover.puml) | sequence | Отказ региона: переключение, fence, проверка и возврат |

## Ключевые решения

1. **Database per Service** — у каждого сервиса свой контур данных (схема/БД, роли, миграции). Физически можно начинать экономно, но логическая изоляция обязательна. Кампании меняет Campaign, деньги — Billing, факты — Statistics.
2. **Тип хранилища под задачу.** PostgreSQL — конфиг, версии, ACID, ledger (Bidding/Campaign/Billing). ClickHouse — поток событий и тяжёлые агрегации (Statistics/Analytics). Redis — горячее состояние/кеш. S3 — байты, архив, WORM.
3. **Горячий путь (bid) без обращений к БД.** Решение из in-memory индекса + Redis; целимся в P95 ≤ 55 мс (при лимите партнёра 80 мс).
4. **Репликация по цене потери.** Sync там, где нельзя терять (конфиг, финансы) — RPO 0. Для статистики/аналитики/Redis достаточно async: восстанавливаемся из Kafka/S3/ledger.
5. **Шардирование не спешим.** ClickHouse и Redis шардируем сразу по `campaign_id` (равномерно + локально для агрегаций/атомарных операций). PostgreSQL пока держим на репликах, кеш и партиции по времени.
6. **Аналитике — ClickHouse read-контур, не реплики Postgres.** Построчный скан миллиарда строк реплики OLTP не спасёт.
7. **CQRS точечно:** Bidding (конфиг → индекс), Statistics (факты → агрегаты), Billing (ledger → read-модель), лёгкий вариант у Campaign.
8. **Кеш — не истина.** Три кластера Redis: `bid`, `rl`, `cache`. Конфиг инвалидируется событием с версией (TTL — страховка), прогрев — из снапшота/S3/реплик.
9. **Kafka — буфер и replay.** Avro + Schema Registry, `acks=all`, RF=3, `minISR=2`, at-least-once + дедуп/outbox. Долгая история — в S3/ClickHouse.
10. **RPO/RTO подтверждаем учениями.** Финансы — RPO 0. RTO: промежуточно ≤ 15 мин, цель 12 мес ≤ 5 мин. Failover гоняем раз в квартал.