@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title C4 Container Diagram — AdScale (TO-BE, горизонт 1 год)

Person(advertiser, "Рекламодатель", "Управляет кампаниями и бюджетами")
Person_Ext(endUser, "Пользователь сайта/приложения", "Видит рекламу, кликает по баннерам")
System_Ext(dspPartner, "DSP/SSP-партнёры", "Обмен bid request / bid response по OpenRTB")
System_Ext(paymentGateway, "Платёжный шлюз", "Пополнение рекламных бюджетов")

System_Boundary(adscale, "AdScale Platform") {

  Container(gateway, "API Gateway", "Envoy", "Аутентификация партнёров, ограничение частоты запросов, маршрутизация, размыкатель, замер времени отклика")

  Container(bidding, "Bidding Service", "C++", "Приём bid request, подбор кандидатов, аукцион, ответ партнёру. Активные кампании и таргетинг — в памяти процесса")
  Container(delivery, "Delivery Service", "Go", "Формирование HTML/JS-разметки баннера и выдача креатива пользователю")
  Container(campaign, "Campaign Service", "Go", "Кампании, параметры таргетинга, настройки. Владелец справочных данных")
  Container(events, "Events Service", "Go", "Сведение показа с результатом аукциона, фильтрация недействительного трафика, журнал событий")
  Container(billing, "Billing Service", "Go", "Списания, счета, балансы, выдача квот бюджета")
  Container(analytics, "Analytics Service", "Python", "Отчётность рекламодателю по агрегатам")
  Container(bff, "Dashboard BFF", "Node.js", "Агрегация данных для кабинета рекламодателя")
  Container(webui, "Advertiser Dashboard", "JavaScript SPA", "Веб-интерфейс кабинета")

  ContainerDb(campaignDb, "Campaign DB", "PostgreSQL", "Кампании, таргетинг, настройки")
  ContainerDb(billingDb, "Billing DB", "PostgreSQL", "Проводки, балансы, счета")
  ContainerDb(eventsDb, "Events DB", "PostgreSQL", "Журнал показов и кликов")
  ContainerDb(olap, "Analytics Store", "ClickHouse", "Агрегаты для отчётности")
  ContainerDb(state, "Shared State", "Redis", "Квоты бюджета, счётчики частоты показов, лимиты запросов")
  ContainerQueue(kafka, "Event Bus", "Apache Kafka", "Потоки событий: показы, клики, изменения кампаний")
}

Rel(dspPartner, gateway, "Bid request / bid response", "HTTPS / OpenRTB")
Rel(gateway, bidding, "Проксирует запросы ставок", "gRPC")
Rel(endUser, delivery, "Запрос креатива, клики", "HTTPS")
Rel(advertiser, webui, "Управляет кампаниями", "HTTPS")
Rel(webui, bff, "Запросы кабинета", "HTTPS / REST")
Rel(bff, campaign, "Чтение и изменение кампаний", "gRPC")
Rel(bff, analytics, "Запрос отчётов", "gRPC")
Rel(bff, billing, "Балансы и счета", "gRPC")
Rel(bff, paymentGateway, "Пополнение бюджета", "HTTPS / REST")

Rel(bidding, state, "Уменьшение квоты бюджета, счётчики частоты", "Redis protocol")
Rel(billing, state, "Выдача и обновление квот", "Redis protocol")

Rel(campaign, kafka, "Публикует изменения кампаний", "Kafka producer")
Rel(kafka, bidding, "Инкрементальные обновления снапшота", "Kafka consumer")
Rel(delivery, kafka, "Публикует показы и клики", "Kafka producer")
Rel(kafka, events, "Поток событий показов и кликов", "Kafka consumer")
Rel(events, kafka, "Публикует события, подлежащие оплате", "Kafka producer")
Rel(kafka, billing, "События для списания", "Kafka consumer")
Rel(kafka, analytics, "События для агрегации", "Kafka consumer")

Rel(campaign, campaignDb, "Чтение и запись", "SQL")
Rel(billing, billingDb, "Проводки и балансы", "SQL")
Rel(events, eventsDb, "Журнал событий", "SQL")
Rel(analytics, olap, "Агрегаты и отчёты", "SQL")
Rel(delivery, campaign, "Данные креатива победителя", "gRPC")

SHOW_LEGEND()
@enduml