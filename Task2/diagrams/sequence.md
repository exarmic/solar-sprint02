
@startuml
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml
title Путь ставки и учёт показа — AdScale (TO-BE)

autonumber
participant "DSP-партнёр" as DSP
participant "API Gateway\n(Envoy)" as GW
participant "Bidding Service" as BID
database "Redis\n(квоты, частота)" as REDIS
participant "Delivery Service" as DEL
queue "Kafka" as KAFKA
participant "Events Service" as EVT
participant "Billing Service" as BILL

== Аукцион: бюджет 60 ms ==
DSP -> GW: POST /openrtb/2.6/bid\n(tmax=80ms)
activate GW
GW -> GW: mTLS, проверка лимита частоты
GW -> BID: Bid(request), deadline=75ms
activate BID
BID -> BID: подбор кандидатов\n(индекс в памяти)
BID -> REDIS: проверка частоты показов
REDIS --> BID: счётчик
BID -> BID: аукцион, расчёт цены
BID -> REDIS: атомарное уменьшение квоты
REDIS --> BID: остаток
alt квота исчерпана / нет кандидатов / истёк дедлайн
  BID --> GW: no-bid
  GW --> DSP: 204 No Content
else ставка сформирована
  BID --> GW: bid response
  deactivate BID
  GW --> DSP: 200 OK (price, crid, nurl)
end
deactivate GW

== Уведомление о выигрыше ==
DSP -> GW: GET /win?auction=...&price=...
GW -> BID: WinNotice
activate BID
BID -> REDIS: SETNX win:{auction_id}
alt первое уведомление
  BID -> KAFKA: AuctionWon (campaign, price)
end
BID --> GW: 200 OK
deactivate BID

== Показ креатива и списание ==
"Пользователь" -> DEL: запрос креатива
activate DEL
DEL --> "Пользователь": HTML/JS баннера
DEL -> DEL: факт показа + outbox\n(одна транзакция)
DEL -> KAFKA: Impression (event_id, bid_id, user_hash)
deactivate DEL

KAFKA -> EVT: Impression
activate EVT
EVT -> EVT: сведение с AuctionWon по bid_id,\nфильтрация недействительного трафика,\nдедупликация по event_id
EVT -> REDIS: INCR freq:{user_hash}:{campaign_id}, TTL сутки
note right of EVT
  Счётчик частоты показов обновляется
  по факту показа, а не по выигрышу:
  выигранная ставка не всегда приводит
  к показу. Задержка в секунды допустима —
  это не финансовое требование.
end note
EVT -> KAFKA: BillableEvent (event_id, сумма)
deactivate EVT

KAFKA -> BILL: BillableEvent
activate BILL
BILL -> BILL: транзакция:\nINSERT проводки (unique event_id)\n+ изменение баланса
note right of BILL
  Конфликт уникального индекса =
  событие уже обработано,
  повторного списания нет.
  Смещение подтверждается
  после коммита транзакции.
end note
deactivate BILL
@enduml