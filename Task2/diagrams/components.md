@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title C4 Component Diagram — Bidding Service

System_Ext(dsp, "DSP-партнёр", "Обмен по OpenRTB")
Container(gateway, "API Gateway", "Envoy", "Аутентификация, лимиты, маршрутизация, размыкатель")
ContainerDb(redis, "Shared State", "Redis", "Квоты бюджета, счётчики частоты, флаги выигрышей")
ContainerDb(campaignDb, "Campaign DB", "PostgreSQL", "Источник кампаний и таргетинга (монолит)")
ContainerQueue(kafka, "Event Bus", "Apache Kafka", "События выигрышей и обновления кампаний")

Container_Boundary(bidding, "Bidding Service") {
  Component(api, "Bid API Handler", "gRPC / HTTP", "Разбор запроса, контроль дедлайна из tmax, формирование ответа или no-bid")
  Component(matcher, "Candidate Matcher", "C++", "Подбор кандидатов по инвертированному индексу таргетинга в памяти")
  Component(rules, "Rule Filter", "C++", "Фильтры: расписание, ограничение частоты, чёрные списки, минимальная цена")
  Component(auction, "Auction Engine", "C++", "Взвешивание ставок, расчёт цены, выбор победителя")
  Component(quota, "Quota Client", "C++", "Атомарное уменьшение квоты бюджета, локальный резерв")
  Component(snapshot, "Snapshot Manager", "C++", "Загрузка снапшота при старте, применение инкрементальных обновлений, готовность экземпляра")
  Component(winh, "Win Notice Handler", "C++", "Обработка уведомления о выигрыше, идемпотентность, публикация события")
  Component(metrics, "Telemetry", "C++", "Распределение времени отклика по участкам, причины no-bid, возраст снапшота")
}

Rel(dsp, gateway, "Bid request / response", "HTTPS / OpenRTB")
Rel(gateway, api, "Bid(request) с дедлайном", "gRPC")
Rel(api, matcher, "Подбор кандидатов")
Rel(matcher, rules, "Кандидаты на фильтрацию")
Rel(rules, auction, "Отфильтрованные кандидаты")
Rel(auction, quota, "Проверка остатка бюджета победителя")
Rel(quota, redis, "Атомарное уменьшение", "RESP")
Rel(rules, redis, "Счётчики частоты показов", "RESP")
Rel(snapshot, campaignDb, "Логическая репликация", "PostgreSQL logical replication")
Rel(snapshot, kafka, "Обновления кампаний (целевое состояние)", "Kafka consumer")
Rel(matcher, snapshot, "Чтение снапшота из памяти")
Rel(gateway, winh, "Win notice", "gRPC")
Rel(winh, redis, "Флаг идемпотентности", "RESP")
Rel(winh, kafka, "AuctionWon", "Kafka producer")
Rel(api, metrics, "Замеры участков")

SHOW_LEGEND()
@enduml