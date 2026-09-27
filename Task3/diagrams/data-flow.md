@startuml
title Потоки данных — AdScale (TO-BE)

skinparam componentStyle rectangle

actor "DSP-партнёр" as DSP
actor "Пользователь" as USER
actor "Рекламодатель" as ADV

component "API Gateway" as GW
component "Bidding\n(снапшот в памяти)" as BID
component "Delivery" as DEL
component "Campaign" as CAMP
component "Events" as EVT
component "Billing" as BILL
component "Analytics" as ANL
component "Dashboard BFF" as BFF

database "Campaign DB\nPostgreSQL" as CDB
database "Billing DB\nPostgreSQL" as BDB
database "Events DB\nPostgreSQL\n(окно дедупликации)" as EDB
database "ClickHouse\n(факты и агрегаты)" as CH
database "Redis\n(квоты, счётчики)" as RDS
queue "Kafka" as K

DSP --> GW : bid request / response
GW --> BID : gRPC
BID --> RDS : квоты, частота (атомарно)
GW --> RDS : сверка счётчиков лимитов

USER --> DEL : запрос креатива
ADV --> BFF : кабинет

CAMP --> CDB : запись мастер-данных
CAMP --> K : campaign.changes (уплотнённый топик)
K --> BID : обновление снапшота
K --> DEL : обновление креативов

BID --> K : auction-won
DEL --> K : impressions, clicks
K --> EVT : факты для сведения
EVT --> EDB : дедупликация, журнал суток
EVT --> K : billable-events
K --> BILL : события для списания
BILL --> BDB : проводки, балансы
BILL --> RDS : выдача квот
K --> ANL : факты для загрузки
ANL --> CH : пакетная загрузка, обезличивание
BFF --> CAMP : чтение/запись кампаний (gRPC)
BFF --> BILL : балансы, счета (gRPC)
BFF --> ANL : отчёты (gRPC)

note right of RDS
  Не кеш: разделяемое
  состояние с атомарными
  операциями
end note

note bottom of CH
  Персональные данные
  обезличиваются
  при загрузке
end note
@enduml