# Task 2 — Проектирование сервисов и взаимодействий

## Состав

| Файл | Содержание | Роль |
|---|---|---|
| [bidding-service.md](bidding-service.md) | Границы сервиса ставок, API (OpenRTB, gRPC, admin), зависимости, модель данных, бюджет задержки, масштабирование | Задача 1 — сервис ставок |
| [interaction.md](interaction.md) | Протоколы внутренних и внешних взаимодействий, обоснование gRPC/REST/Kafka, контракты, паттерны надёжности | Задачи 2 и 4 — взаимодействия и надёжность |
| [api-gateway.md](api-gateway.md) | Дизайн API Gateway: маршрутизация, лимиты, аутентификация, circuit breaker, мониторинг отклика | Задача 3 — точка входа для DSP |
| [diagrams/](diagrams/) | C4 (3 шт.) и sequence (6 шт.) в формате PlantUML — все схемы вынесены отдельно | Схемы |

## Диаграммы

| Файл | Тип | Что показывает |
|---|---|---|
| [diagrams/C4_Component_Bidding.puml](diagrams/C4_Component_Bidding.puml) | C4 component | Компоненты сервиса ставок и зависимости |
| [diagrams/C4_Component_ApiGateway.puml](diagrams/C4_Component_ApiGateway.puml) | C4 component | Компоненты API Gateway для DSP |
| [diagrams/C4_Deployment_EdgeTopology.puml](diagrams/C4_Deployment_EdgeTopology.puml) | C4 deployment | Edge-топология: GSLB, 3 AZ, пути bid и beacon |
| [diagrams/Sequence_Bid_HappyPath.puml](diagrams/Sequence_Bid_HappyPath.puml) | sequence | Нормальный путь bid request с бюджетом 55 мс |
| [diagrams/Sequence_Event_ImpressionClick.puml](diagrams/Sequence_Event_ImpressionClick.puml) | sequence | Поток событий показа и клика, дедупликация |
| [diagrams/Sequence_Config_Propagation.puml](diagrams/Sequence_Config_Propagation.puml) | sequence | Доставка конфигурации и kill switch |
| [diagrams/Sequence_Degradation_CircuitBreaker.puml](diagrams/Sequence_Degradation_CircuitBreaker.puml) | sequence | Breaker, fallback и быстрый 204 |
| [diagrams/Sequence_Gateway_AuthRateLimit.puml](diagrams/Sequence_Gateway_AuthRateLimit.puml) | sequence | Аутентификация, квоты, breaker, SLA-метрики |
| [diagrams/Sequence_Billing_Retry_Idempotency.puml](diagrams/Sequence_Billing_Retry_Idempotency.puml) | sequence | Повторы с backoff и идемпотентное списание |

## Ключевые решения

1. Сервис ставок — один bounded context и две единицы развёртывания: Bid Edge на Go отвечает за протокол и ввод-вывод, Auction Core на C++ переиспользует существующий движок. Чтений базы в горячем пути внутри сервиса нет ни одного.
2. В горячем пути gRPC, в control plane REST, события идут через Kafka. Асимметрия выбрана по профилю нагрузки, а не по вкусу: 55 мс бюджета не прощают JSON-сериализации на трёх хопах, а 30 RPS управления от protobuf ничего не выигрывают.
3. Наружу отдаём OpenRTB 2.6 поверх HTTPS — единственный реалистичный вариант внешней интеграции. Формат партнёра изолирован антикоррупционным слоем на gateway и Bid Edge, чтобы его изменения не расходились по коду.
4. API Gateway построен на Envoy с rate limiting через Redis: breaker и лимиты есть из коробки, за запрос платить не нужно, а переход на Istio не потребует менять продукт.
5. Надёжность держится на четырёх вещах: breaker на двух уровнях, повторы только вне горячего пути, идемпотентность по детерминированным ключам и лестница fallback вплоть до `204`. Отвечать `5xx` как режимом деградации мы сознательно не стали.