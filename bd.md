**Общая подготовка:**
- Экземпляр SQL Server 2025 (Developer Edition).
- Скрипт `00_setup.sql`, который создаёт БД `ShopDB` со схемами `app`, `rpt`, `sec`, `audit` и таблицами:
  - `app.Customers(CustomerID, TenantID, Region, Name, Email, CreatedAt)`
  - `app.Orders(OrderID, CustomerID, OrderDate, Status, Total)`
  - `app.OrderDetails(OrderDetailID, OrderID, ProductID, Qty, Price)`
  - `app.Products(ProductID, CategoryID, Name, Price)`
  - `app.Categories(CategoryID, ParentID, Name)`
  - `app.Payments(PaymentID, OrderID, PaidAt, Amount)`
  - `audit.OrderLog(LogID, OrderID, Action, ChangedAt, ChangedBy)`
- Наполнение: ~50 000 заказов, ~200 000 позиций, ~5 000 клиентов, ~500 товаров, дерево категорий глубиной 5.
- Каждый билет работает со своей копией БД: `ShopDB_T01` … `ShopDB_T15`.

**Формат билета:** 5 заданий, 90 минут, отчёт — скрипты + вывод + пояснения.

---

## Билет 1. «Стенд с нуля»

**Подготовка:** развернуть SQL Server 2025, восстановить `ShopDB_T01` из бэкапа.

**Задание 1.** Аудит схемы и CTE.
- Проверить `sys.databases`, `sys.tables`, `sys.columns` по схеме `app`.
- Найти несогласованные типы (`NVARCHAR` vs `VARCHAR`, `DATETIME` vs `DATETIME2`), лишние `NULL`, отсутствующие FK.
- Переписать построчный подсчёт «сумма заказов по клиенту» из `WHILE` в один запрос с CTE и `GROUP BY`.

**Задание 2.** Оконные функции, APPLY, представление, процедура, функция.
- Через `ROW_NUMBER()` и `SUM() OVER(PARTITION BY CustomerID ORDER BY OrderDate)` получить накопительный итог по каждому клиенту.
- Через `CROSS APPLY` вывести последние 2 заказа каждого клиента.
- Создать `rpt.vw_CustomerOrderSummary` со `SCHEMABINDING`.
- Создать `app.usp_GetCustomerOrders @CustomerID, @Count OUTPUT`.
- Создать скалярную `app.fn_OrderTotal(@OrderID)` и показать её проблему в `WHERE`.

**Задание 3.** Триггер, транзакция, изоляция, locks, deadlock.
- Создать `app.trg_OrderLog` (`AFTER INSERT, UPDATE, DELETE`) на `app.Orders` с записью в `audit.OrderLog`, проверить multi-row.
- Обернуть многотабличное обновление заказа в `BEGIN TRAN / TRY / CATCH / XACT_STATE()`.
- В сессии A открыть транзакцию и обновить строку; в сессии B прочитать ту же строку в `READ COMMITTED` — зафиксировать блокировку.
- Найти блокирующую сессию через `sp_who2` и `sys.dm_tran_locks`.
- Воспроизвести deadlock между `app.Orders` и `app.OrderDetails`, поймать ошибку 1205, разобрать граф.

**Задание 4.** Планы, индексы, SARGability, Query Store, tempdb.
- Включить actual plan, сравнить scan и seek по `app.Orders`.
- Создать clustered по `OrderID`, nonclustered по `OrderDate`, composite `(CustomerID, OrderDate)`, covering и filtered `WHERE Status='Active'`.
- Показать не-SARGable `WHERE YEAR(OrderDate)=2025` vs SARGable `OrderDate >= '2025-01-01' AND OrderDate < '2026-01-01'`.
- Включить Query Store, найти регрессию после добавления индекса.
- Сравнить `#temp` и `@table`, выполнить пакетную вставку 100 000 строк в `#temp`.

**Задание 5.** Безопасность, RLS, аудит, backup.
- Создать логин `app_login`, пользователя в `ShopDB_T01`, роли `db_datareader/db_datawriter`, схему `app`.
- Включить RLS по `TenantID` на `app.Orders`.
- Маскировать `app.Customers.Email` через `MASKED WITH`.
- Включить Server Audit на `SELECT` к `app.Customers`.
- Перевести БД в `FULL`, сделать full + differential + log backup, показать backup chain.

---

## Билет 2. «Аудит после ADO.NET»

**Подготовка:** восстановить `ShopDB_T02`, включить `READ_COMMITTED_SNAPSHOT ON`.

**Задание 1.**
- Найти в `app.Orders` колонки с `DATETIME` вместо `DATETIME2`, `VARCHAR` вместо `NVARCHAR`, `NULL` там, где ожидается `NOT NULL`.
- Найти отсутствующие FK между `app.OrderDetails` и `app.Products`.
- Заменить курсор по заказам на CTE + `MERGE` в `app.OrderDetails`.

**Задание 2.**
- `RANK()` и `LAG()` для сравнения суммы заказов по месяцам у каждого клиента.
- `OUTER APPLY` для последнего платежа по заказу.
- Создать `rpt.vw_AppCustomer` как стабильный контракт.
- Создать `app.usp_UpsertCustomer` с `OUTPUT @Result` и error contract.
- Создать inline TVF `app.fn_OrdersByStatus(@Status)` и сравнить с scalar по плану.

**Задание 3.**
- `INSTEAD OF UPDATE` триггер на `rpt.vw_AppCustomer` с multi-row обработкой.
- Транзакция с `SAVE TRANSACTION` и откатом к точке сохранения.
- Воспроизвести фантомное чтение при `READ COMMITTED SNAPSHOT`.
- Найти транзакцию, блокирующую `ALTER TABLE app.Orders`.
- Снять граф deadlock через Extended Events (`system_health`), исправить порядок доступа.

**Задание 4.**
- Сравнить estimated и actual plan, оценить cardinality estimate.
- Сравнить clustered и heap на `app.OrderDetails`.
- Создать filtered index `WHERE Status='Active'`, покрыть запрос.
- В Query Store найти регрессию после `UPDATE STATISTICS`.
- Выполнить `MERGE` большой таблицы пакетами, следить за tempdb.

**Задание 5.**
- Создать схему `app`, роль `app_executor`, grant execute на процедуры.
- RLS по `Region` на `app.Customers`.
- Audit на `UPDATE` к `app.Customers`, DDM на `Email`.
- Full + log backup, восстановление на точку `STOPAT`.

---

## Билет 3. «CTE и set-based мышление»

**Подготовка:** восстановить `ShopDB_T03`.

**Задание 1.**
- Проверить типы и ограничения в `app.Categories`.
- Рекурсивный CTE для дерева категорий (глубина 5).
- Заменить построчный обход дерева в `WHILE` на рекурсивный CTE.

**Задание 2.**
- `NTILE(4)` для квартилей продаж по клиентам.
- `CROSS APPLY` топ-3 товара по каждому клиенту.
- Создать `rpt.vw_CategoryTree` со `SCHEMABINDING`.
- Создать `rpt.usp_ReportSales @From, @To, @Total OUTPUT`.
- Сравнить рекурсивную scalar и inline TVF по плану.

**Задание 3.**
- `AFTER DELETE` триггер на `app.Orders`, архивирующий удалённые строки, multi-row.
- `TRY/CATCH` + `XACT_STATE()` в транзакции.
- Сравнить `SERIALIZABLE` и `SNAPSHOT`, воспроизвести фантом.
- Мониторинг через `sys.dm_os_waiting_tasks`.
- Deadlock между `app.Categories` и `app.Products`, анализ графа.

**Задание 4.**
- План рекурсивного CTE, оценка cost.
- Индекс на `ParentID` для рекурсии.
- SARGable vs non-SARGable в рекурсивном CTE.
- Query Store: сравнить планы до/после индекса.
- Рекурсивный CTE и tempdb spill, пакетная обработка.

**Задание 5.**
- Логин `report_user`, роль `db_datareader`, схема `rpt`.
- RLS на `rpt.vw_CategoryTree` по подразделению.
- Server Audit на чтение дерева категорий.
- Full + diff backup, восстановить diff, проверить целостность.

---

## Билет 4. «Оконные функции и APPLY»

**Подготовка:** восстановить `ShopDB_T04`.

**Задание 1.**
- Аудит `app.Orders` и `app.OrderDetails` — типы, ограничения, индексы.
- CTE для агрегатов по клиентам вместо вложенных подзапросов.

**Задание 2.**
- `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG/LEAD`.
- `CROSS APPLY` и `OUTER APPLY` для связанных выборок.
- `rpt.vw_SalesRanked` со `SCHEMABINDING`.
- `app.usp_TopNCustomers @N, @List OUTPUT`.
- Inline TVF `app.fn_OrdersByDateRange(@From,@To)`.

**Задание 3.**
- `AFTER UPDATE` триггер на `app.Orders` при смене `Status`, multi-row.
- Транзакция с `SAVE TRANSACTION` и обработкой ошибок.
- `READ COMMITTED` vs `SNAPSHOT`, воспроизвести неповторяющееся чтение.
- Найти блокировку при DDL на `app.Orders`.
- Deadlock через Extended Events.

**Задание 4.**
- Влияние оконных функций на план (sort, spool).
- Индекс `(CustomerID, OrderDate)` под оконные функции.
- SARGable фильтр по датам, filtered index.
- Query Store: регрессия после изменения запроса.
- Оконные функции и сортировка в tempdb, пакетная вставка.

**Задание 5.**
- Роль `analyst`, grant select на `rpt.vw_SalesRanked`.
- RLS по менеджеру на `app.Orders`.
- Audit на `SELECT` из представления.
- Log backup, восстановление на точку.

---

## Билет 5. «Представления как контракты»

**Подготовка:** восстановить `ShopDB_T05`.

**Задание 1.**
- Аудит колонок и типов `app.Customers`, `app.Orders`.
- Найти нарушения контракта (переименования, смена типов).
- CTE для подготовки данных представления.

**Задание 2.**
- `ROW_NUMBER()` для последней версии записи в представлении.
- `rpt.vw_AppContract` со `SCHEMABINDING`.
- `app.usp_GetContract @CustomerID, @Result OUTPUT`.
- Inline TVF для фильтрации представления.

**Задание 3.**
- `INSTEAD OF INSERT` триггер на `rpt.vw_AppContract` для записи в `app.Customers`.
- Транзакция вокруг записи через представление.
- `SNAPSHOT` при чтении представления во время обновления.
- Блокировка при `SCHEMABINDING`.
- Deadlock через представление и базовую таблицу.

**Задание 4.**
- Расширение представления в плане, оценка cost.
- Indexed view (clustered) для агрегатов по `app.Orders`.
- SARGability при фильтрации по indexed view.
- Query Store: регрессия после изменения представления.
- Indexed view и tempdb, пакетная вставка.

**Задание 5.**
- Grant select на представление, deny на базовые таблицы.
- RLS через представление.
- Audit на доступ к представлению.
- Full + log backup, восстановление и проверка indexed view.

---

## Билет 6. «Хранимые процедуры как API»

**Подготовка:** восстановить `ShopDB_T06`.

**Задание 1.**
- Аудит параметров и типов в существующих процедурах (если есть).
- CTE внутри процедуры для агрегатов.

**Задание 2.**
- `ROW_NUMBER` внутри процедуры.
- `rpt.vw_ProcedureInput` для входных данных.
- `app.usp_ProcessOrder @OrderID, @Result OUTPUT, @ErrCode OUTPUT`.
- Inline TVF внутри процедуры.

**Задание 3.**
- Триггер на `app.OrderDetails`, multi-row.
- `TRY/CATCH`, `XACT_STATE`, откат.
- `READ COMMITTED` vs `SNAPSHOT` внутри процедуры.
- Блокировка при вызове процедуры.
- Deadlock при двух одновременных вызовах процедуры.

**Задание 4.**
- План процедуры, parameter sniffing.
- Индексы под процедуру.
- SARGable условия в процедуре.
- Query Store: история выполнения процедуры, регрессия.
- Temp-таблицы внутри процедуры, пакетная обработка.

**Задание 5.**
- Grant execute на процедуру, deny на таблицы.
- RLS внутри процедуры.
- Audit на вызов процедуры.
- Full + log backup, восстановление и проверка процедуры.

---

## Билет 7. «Пользовательские функции»

**Подготовка:** восстановить `ShopDB_T07`.

**Задание 1.**
- Аудит типов и ограничений, поиск дублирования логики в функциях.
- CTE внутри inline TVF.

**Задание 2.**
- Оконные функции в inline TVF.
- Представление поверх TVF.
- Процедура вызывает TVF и возвращает `OUTPUT`.
- Scalar vs inline TVF vs multi-statement TVF, сравнить планы.

**Задание 3.**
- Триггер вызывает TVF.
- Транзакция вокруг вызова функции, `TRY/CATCH`.
- `SNAPSHOT` при вызове TVF.
- Блокировка при вызове функции.
- Deadlock с функцией и таблицей.

**Задание 4.**
- Оценка cost scalar vs inline.
- Индекс под TVF.
- SARGability при вызове TVF.
- Query Store: регрессия после изменения TVF.
- Tempdb spill в TVF, пакетный вызов.

**Задание 5.**
- Grant select на TVF.
- RLS через TVF.
- Audit на вызов TVF.
- Full + diff backup, восстановление.

---

## Билет 8. «Триггеры и целостность»

**Подготовка:** восстановить `ShopDB_T08`.

**Задание 1.**
- Аудит ограничений, поиск мест без FK.
- CTE внутри триггера для set-based обработки.

**Задание 2.**
- Оконные функции в триггере.
- Представление для аудита.
- Процедура как альтернатива триггеру.
- TVF для проверки целостности.

**Задание 3.**
- `AFTER INSERT/UPDATE/DELETE`, `inserted/deleted`, multi-row.
- Транзакция внутри триггера, `TRY/CATCH`.
- `SNAPSHOT` при работе триггера.
- Триггер блокирует `app.Orders`, найти сессию.
- Deadlock: триггер и внешняя транзакция.

**Задание 4.**
- План триггера.
- Индексы для триггера.
- SARGable условия в триггере.
- Query Store: регрессия из-за триггера.
- Триггер и tempdb, пакетная вставка.

**Задание 5.**
- Deny на прямое изменение `app.Orders`, только через процедуру.
- RLS в триггере.
- Audit на срабатывание триггера.
- Full + log backup, восстановление.

---

## Билет 9. «Транзакции T-SQL»

**Подготовка:** восстановить `ShopDB_T09`.

**Задание 1.**
- Аудит типов, ограничений, транзакционных границ.
- CTE в транзакционном блоке.

**Задание 2.**
- Оконные функции в транзакции.
- Представление для отчёта по транзакциям.
- Процедура с `TRY/CATCH`, `XACT_STATE`.
- TVF внутри транзакции.

**Задание 3.**
- `BEGIN/COMMIT/ROLLBACK`, `SAVE TRANSACTION`, `XACT_STATE`.
- Уровни `READ UNCOMMITTED`, `READ COMMITTED`, `REPEATABLE READ`, `SERIALIZABLE`, `SNAPSHOT`.
- Найти блокирующую транзакцию.
- Deadlock: два сеанса, разный порядок, граф.

**Задание 4.**
- План транзакционного запроса.
- Индексы для транзакций.
- SARGable в транзакции.
- Query Store: отслеживание регрессии.
- Транзакция и tempdb, пакетная обработка.

**Задание 5.**
- Grant execute на процедуру.
- RLS в транзакции.
- Audit на транзакции.
- Log backup, восстановление на точку.

---

## Билет 10. «Изоляция и аномалии»

**Подготовка:** восстановить `ShopDB_T10`.

**Задание 1.**
- Аудит схемы, типы, ограничения.
- CTE для воспроизведения аномалий.

**Задание 2.**
- Оконные функции для анализа аномалий.
- Представление для отчёта по аномалиям.
- Процедура с параметром уровня изоляции.
- TVF для анализа.

**Задание 3.**
- `SET TRANSACTION ISOLATION LEVEL`.
- Грязное, неповторяющееся, фантомное чтение.
- `sys.dm_tran_locks`.
- Граф deadlock через Extended Events.

**Задание 4.**
- План при разных уровнях.
- Влияние индексов на изоляцию.
- SARGable при разных уровнях.
- Query Store: регрессия при смене уровня.
- Row versioning в tempdb.

**Задание 5.**
- Роли для тестирования.
- RLS при разных уровнях.
- Audit на аномалии.
- Full + log backup.

---

## Билет 11. «Locks и blocking»

**Подготовка:** восстановить `ShopDB_T11`.

**Задание 1.**
- Аудит схемы.
- CTE для анализа блокировок.

**Задание 2.**
- Оконные функции для мониторинга.
- Представление для отчёта по блокировкам.
- Процедура-монитор блокировок.
- TVF для парсинга DMV.

**Задание 3.**
- Длинная транзакция на `app.Orders`.
- Блокировки при разных уровнях.
- `sp_who2`, `sys.dm_exec_requests`, `sys.dm_tran_locks`.
- Граф deadlock и `system_health`.

**Задание 4.**
- План блокирующего запроса.
- Индексы снижают блокировки.
- SARGable снижает блокировки.
- Query Store: регрессия из-за блокировок.
- Tempdb при блокировках.

**Задание 5.**
- Роль для мониторинга.
- RLS и блокировки.
- Audit на блокировки.
- Full + log backup.

---

## Билет 12. «Deadlock»

**Подготовка:** восстановить `ShopDB_T12`.

**Задание 1.**
- Аудит схемы.
- CTE для анализа графа.

**Задание 2.**
- Оконные функции для анализа.
- Представление для отчёта по deadlock.
- Процедура, провоцирующая deadlock.
- TVF для разбора XML графа.

**Задание 3.**
- Два сеанса, разный порядок доступа к `app.Orders` и `app.OrderDetails`.
- Влияние уровня изоляции.
- Мониторинг блокировок.
- Граф через Extended Events, ошибка 1205.

**Задание 4.**
- План участников.
- Индексы снижают deadlock.
- SARGable снижает deadlock.
- Query Store: регрессия после исправления.
- Tempdb при deadlock.

**Задание 5.**
- Роль для анализа.
- RLS и deadlock.
- Audit на deadlock.
- Full + log backup.

---

## Билет 13. «Execution Plan»

**Подготовка:** восстановить `ShopDB_T13`.

**Задание 1.**
- Аудит схемы.
- CTE в анализируемом запросе.

**Задание 2.**
- Оконные функции в плане.
- Представление и его план.
- Процедура и parameter sniffing.
- TVF и план.

**Задание 3.**
- Триггер и план.
- План транзакции.
- План при разных уровнях.
- План и блокировки.
- План участников deadlock.

**Задание 4.**
- Estimated vs actual, cardinality, statistics.
- Clustered vs nonclustered в плане.
- SARGable и seek.
- Query Store: план-регрессия.
- Spill в плане (tempdb).

**Задание 5.**
- Grant showplan.
- RLS в плане.
- Audit на анализ планов.
- Full + log backup.

---

## Билет 14. «Индексы и SARGability»

**Подготовка:** восстановить `ShopDB_T14`.

**Задание 1.**
- Аудит схемы и индексов.
- CTE с индексами.

**Задание 2.**
- Оконные функции и индексы.
- Indexed view.
- Процедура с индексом.
- TVF и индекс.

**Задание 3.**
- Триггер и индекс.
- Транзакция и индекс.
- Изоляция и индекс.
- Индекс снижает блокировки.
- Индекс снижает deadlock.

**Задание 4.**
- Seek vs scan.
- Clustered, nonclustered, composite, covering, filtered.
- SARGable vs non-SARGable, примеры.
- Query Store: регрессия после удаления индекса.
- Индекс и tempdb.

**Задание 5.**
- Grant на индекс.
- RLS и индекс.
- Audit на изменение индексов.
- Full + log backup, восстановление индексов.

---

## Билет 15. «Итоговый комплексный»

**Подготовка:** развернуть экземпляр с нуля, восстановить `ShopDB_T15`, снимок.

**Задание 1.**
- Полный аудит типов, ограничений, границ.
- Рекурсивный CTE + set-based преобразование.

**Задание 2.**
- Полный набор оконных функций и APPLY.
- `rpt.vw_AppContract` со `SCHEMABINDING`.
- `app.usp_Main` с `OUTPUT` и error contract.
- Scalar vs inline TVF, сравнение.

**Задание 3.**
- `AFTER` триггер с multi-row.
- `TRY/CATCH`, `XACT_STATE`, `SAVE TRANSACTION`.
- Все уровни изоляции, воспроизведение аномалий.
- Диагностика живой системы.
- Воспроизведение и исправление deadlock.

**Задание 4.**
- Estimated vs actual, cardinality, statistics.
- Clustered + nonclustered.
- Composite, covering, filtered, SARGable.
- Query Store: история и регрессия.
- Temp-структуры, пакетная обработка.

**Задание 5.**
- Logins, users, schemas, roles, permissions.
- Least privilege + RLS.
- Server Audit + DDM.
- Full + diff + log backup, восстановление на точку.

---

## Регламент билета (90 минут, 5 заданий)

| Задание | Темы |
|---------|------|
| 1 | Стенд, аудит схемы, CTE, set-based |
| 2 | Оконные функции, APPLY, представления, процедуры, функции |
| 3 | Триггеры, транзакции, изоляция, locks, deadlock |
| 4 | Execution Plan, индексы, SARGability, Query Store, tempdb |
| 5 | Безопасность, least privilege, RLS, аудит, backup/recovery |

**Критерии оценки:** работающий стенд, корректные T-SQL-скрипты, воспроизведённые сценарии блокировок и deadlock, разобранные планы, настроенная безопасность, рабочий backup chain, письменный отчёт с пояснениями.
