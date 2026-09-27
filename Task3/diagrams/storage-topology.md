@startuml
title Топология хранилищ и отказоустойчивость — AdScale

package "Зона доступности A" {
  database "Billing DB\nведущий" as B1
  database "Campaign DB\nведущий" as C1
  database "Events DB\nведущий" as E1
  database "Redis\nведущий" as R1
  node "Kafka broker 1" as K1
  database "ClickHouse\nшард 1, реплика 1" as CH1
}

package "Зона доступности B" {
  database "Billing DB\nсинхронная реплика" as B2
  database "Campaign DB\nсинхронная реплика" as C2
  database "Events DB\nасинхронная реплика" as E2
  database "Redis\nреплика" as R2
  node "Kafka broker 2" as K2
  database "ClickHouse\nшард 1, реплика 2" as CH2
}

package "Зона доступности C" {
  node "Kafka broker 3" as K3
  database "ClickHouse\nшард 2, реплика 1" as CH3
}

B1 -[#red]-> B2 : синхронная репликация\nRPO = 0
C1 -[#red]-> C2 : синхронная репликация\nRPO = 0
E1 -[#blue]-> E2 : асинхронная репликация\nRPO = 5 мин
R1 -[#blue]-> R2 : асинхронная репликация\nдопускается потеря
K1 -[#green]-> K2 : replication.factor = 3
K2 -[#green]-> K3
CH1 -[#blue]-> CH2 : реплицированные таблицы

cloud "Хранилище резервных копий\n(вне зон доступности)" as BKP
B1 ..> BKP : ежесуточная копия + журнал предзаписи
C1 ..> BKP : ежесуточная копия + журнал предзаписи
E1 ..> BKP : ежесуточная копия
CH1 ..> BKP : еженедельная копия

note right of BKP
  Восстановление проверяется
  ежемесячно для Billing,
  ежеквартально для остальных
end note
@enduml