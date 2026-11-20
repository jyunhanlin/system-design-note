# system-design-note

System design 學習筆記：耦合、可靠性、分散式系統與架構邊界。

## Index

- [Hidden Coupling](./hidden-coupling.md)：架構圖看不到的依賴，以及五個檢視面向與六種類型
- [Observability Basics](./observability-basics.md)：logs / metrics / traces 是同一串 event 的三種投影，以及 cardinality、sampling、alerting、dashboards
- [Between Services](./between-services.md)：架構圖箭頭裡的系統，包括 timeout ≠ 失敗、deadline 預算、retry 相乘、connection pool 耗盡、DNS 與連線生命週期
