# 15 билетов по ADO.NET + PostgreSQL

**Общие условия для всех билетов:**
- Время: 1,5 часа
- БД: PostgreSQL (Docker)
- Проект: ASP.NET Core Minimal API (.NET 8), пакеты `Npgsql` и `Dapper`
- В **каждом** билете студент должен реализовать **все темы**: соединение, команды, параметры, транзакции, DataSet, DataAdapter, DataTable, асинхронность, пулы, Dapper
- Каркасы содержат только сигнатуры эндпоинтов и пустые тела — без подсказок и решений

---

## Общие файлы (для всех билетов)

**`MyApp.csproj`**
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <AssemblyName>MyApp</AssemblyName>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Npgsql" Version="8.0.5" />
    <PackageReference Include="Dapper" Version="2.1.35" />
  </ItemGroup>
</Project>
```

**`Dockerfile`**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app .
ENV ASPNETCORE_URLS=http://+:80
EXPOSE 80
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**`docker-compose.yml`**
```yaml
version: '3.8'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: shopdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
  app:
    build: .
    environment:
      CONN_STRING: "Host=db;Port=5432;Database=shopdb;Username=postgres;Password=postgres;Pooling=true;Minimum Pool Size=1;Maximum Pool Size=20"
    ports: ["8080:80"]
    depends_on: [db]
```

---

## Билет 1 — «Интернет-магазин: каталог товаров»

**Задание:** Реализовать эндпоинты:
- `GET /products?min=` — список товаров с фильтром по минимальной цене (соединение + команда + параметры + async + пул);
- `GET /products/dapper?min=` — тот же список через Dapper;
- `GET /products/dataset` — отчёт через `DataSet`/`DataAdapter`/`DataTable` с фильтрацией `DataTable.Select`;
- `POST /orders` — оформление заказа в транзакции.

**`init.sql`**
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price NUMERIC(10,2) NOT NULL
);
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id),
    qty INT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
INSERT INTO products (name, price) VALUES
    ('Ноутбук', 75000.00),
    ('Мышь', 1500.00),
    ('Клавиатура', 3500.00);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapGet("/products", async (decimal min) =>
{
    return Results.Ok();
});

app.MapGet("/products/dapper", async (decimal min) =>
{
    return Results.Ok();
});

app.MapGet("/products/dataset", () =>
{
    return Results.Ok();
});

app.MapPost("/orders", async (OrderReq req) =>
{
    return Results.Ok();
});

app.Run();
record OrderReq(int ProductId, int Qty);
```

---

## Билет 2 — «Банк: счета и переводы»

**Задание:** Эндпоинты:
- `POST /transfer` — перевод в транзакции с `FOR UPDATE` (соединение + команда + параметры + транзакция + async + пул);
- `GET /accounts` — список через Dapper;
- `GET /accounts/report` — отчёт через `DataSet`/`DataAdapter`/`DataTable`.

**`init.sql`**
```sql
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    owner VARCHAR(100) NOT NULL,
    balance NUMERIC(12,2) NOT NULL DEFAULT 0
);
INSERT INTO accounts (owner, balance) VALUES ('Alice', 1000.00), ('Bob', 500.00);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapPost("/transfer", async (Transfer t) =>
{
    return Results.Ok();
});

app.MapGet("/accounts", async () =>
{
    return Results.Ok();
});

app.MapGet("/accounts/report", () =>
{
    return Results.Ok();
});

app.Run();
record Transfer(int FromId, int ToId, decimal Amount);
```

---

## Билет 3 — «Склад: остатки и приход»

**Задание:** Эндпоинты:
- `GET /inventory/search?name=` — безопасный поиск через параметр `ILIKE` (соединение + команда + параметры + async + пул);
- `POST /inventory/income` — массовый приход в транзакции с логированием;
- `GET /inventory/dapper` — выборка через Dapper;
- `GET /inventory/table` — отчёт через `DataSet`/`DataAdapter`/`DataTable`.

**`init.sql`**
```sql
CREATE TABLE inventory (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    qty INT NOT NULL DEFAULT 0
);
CREATE TABLE movements (
    id SERIAL PRIMARY KEY,
    inventory_id INT REFERENCES inventory(id),
    delta INT NOT NULL,
    at TIMESTAMP DEFAULT NOW()
);
INSERT INTO inventory (name, qty) VALUES ('Гвозди', 100), ('Шурупы', 50);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapGet("/inventory/search", async (string name) =>
{
    return Results.Ok();
});

app.MapPost("/inventory/income", async (Income[] items) =>
{
    return Results.Ok();
});

app.MapGet("/inventory/dapper", async () =>
{
    return Results.Ok();
});

app.MapGet("/inventory/table", () =>
{
    return Results.Ok();
});

app.Run();
record Income(int Id, int Delta);
```

---

## Билет 4 — «Библиотека: книги и выдачи»

**Задание:** Эндпоинты:
- `POST /loans` — выдача в транзакции с проверкой копий (соединение + команда + параметры + транзакция + async + пул);
- `GET /books/search?q=` — поиск через Dapper;
- `GET /loans/report` — две таблицы в `DataSet`, связанные через `DataRelation` (`DataAdapter` + `DataTable`).

**`init.sql`**
```sql
CREATE TABLE books (
    id SERIAL PRIMARY KEY,
    title VARCHAR(150) NOT NULL,
    copies INT NOT NULL
);
CREATE TABLE loans (
    id SERIAL PRIMARY KEY,
    book_id INT REFERENCES books(id),
    reader VARCHAR(100) NOT NULL
);
INSERT INTO books (title, copies) VALUES
    ('Война и мир', 3),
    ('Преступление и наказание', 2);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapPost("/loans", async (LoanReq l) =>
{
    return Results.Ok();
});

app.MapGet("/books/search", async (string q) =>
{
    return Results.Ok();
});

app.MapGet("/loans/report", () =>
{
    return Results.Ok();
});

app.Run();
record LoanReq(int BookId, string Reader);
```

---

## Билет 5 — «HR: сотрудники и отделы»

**Задание:** Эндпоинты:
- `GET /parallel` — параллельно `Task.WhenAll` получить count(сотрудники) и count(отделы) с замером времени (соединение + команда + async + пул);
- `POST /raise` — повышение зарплаты отдела в транзакции с параметрами;
- `GET /employees` — через Dapper;
- `GET /employees/table` — отчёт через `DataSet`/`DataAdapter`/`DataTable`.

**`init.sql`**
```sql
CREATE TABLE departments (id SERIAL PRIMARY KEY, name VARCHAR(100) NOT NULL);
CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    dep_id INT REFERENCES departments(id),
    name VARCHAR(100) NOT NULL,
    salary NUMERIC(10,2) NOT NULL
);
INSERT INTO departments (name) VALUES ('IT'), ('HR');
INSERT INTO employees (dep_id, name, salary) VALUES
    (1, 'Иван', 80000),
    (2, 'Мария', 70000);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using System.Diagnostics;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapGet("/parallel", async () =>
{
    return Results.Ok();
});

app.MapPost("/raise", async (Raise r) =>
{
    return Results.Ok();
});

app.MapGet("/employees", async () =>
{
    return Results.Ok();
});

app.MapGet("/employees/table", () =>
{
    return Results.Ok();
});

app.Run();
record Raise(int DepId, decimal Delta);
```

---

## Билет 6 — «Логистика: доставки»

**Задание:** Эндпоинты:
- `GET /pool-compare` — сравнить время 5 открытий соединения с `Pooling=true` и `Pooling=false`;
- `POST /shipments/close` — закрытие доставки в транзакции с параметром;
- `GET /shipments/dapper` — через Dapper;
- `GET /shipments/table` — через `DataSet`/`DataAdapter`/`DataTable`.

**`init.sql`**
```sql
CREATE TABLE shipments (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'new'
);
INSERT INTO shipments (code, status) VALUES ('A1','new'), ('B2','new');
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using System.Diagnostics;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapGet("/pool-compare", async () =>
{
    return Results.Ok();
});

app.MapPost("/shipments/close", async (int id) =>
{
    return Results.Ok();
});

app.MapGet("/shipments/dapper", async () =>
{
    return Results.Ok();
});

app.MapGet("/shipments/table", () =>
{
    return Results.Ok();
});

app.Run();
```

---

## Билет 7 — «Форум: посты и комментарии»

**Задание:** Эндпоинты:
- `GET /posts/datatable` — создать `DataTable` вручную, заполнить из `NpgsqlDataReader` (соединение + команда + async + пул) и отфильтровать `Select`;
- `POST /posts` — создание поста в транзакции с параметром;
- `GET /comments?postId=` — через Dapper;
- `GET /comments/dataset` — через `DataSet`/`DataAdapter`.

**`init.sql`**
```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL
);
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INT REFERENCES posts(id),
    body TEXT NOT NULL
);
INSERT INTO posts (title) VALUES ('ADO.NET'), ('PostgreSQL');
INSERT INTO comments (post_id, body) VALUES (1,'good'), (1,'nice'), (2,'ok');
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapGet("/posts/datatable", async () =>
{
    return Results.Ok();
});

app.MapPost("/posts", async (string title) =>
{
    return Results.Ok();
});

app.MapGet("/comments", async (int postId) =>
{
    return Results.Ok();
});

app.MapGet("/comments/dataset", () =>
{
    return Results.Ok();
});

app.Run();
```

---

## Билет 8 — «Аптека: лекарства и продажи»

**Задание:** Эндпоинты:
- `GET /meds?q=` — Dapper с `QueryMultiple` (соединение + пул + параметры);
- `POST /sale` — продажа в транзакции: `ExecuteScalar` + `ExecuteNonQuery` (команда + параметры + транзакция + async);
- `GET /meds/report` — `DataSet`/`DataAdapter`/`DataTable`.

**`init.sql`**
```sql
CREATE TABLE meds (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    qty INT NOT NULL,
    price NUMERIC(10,2) NOT NULL
);
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    med_id INT REFERENCES meds(id),
    qty INT NOT NULL,
    at TIMESTAMP DEFAULT NOW()
);
INSERT INTO meds (name, qty, price) VALUES ('Аспирин', 100, 50), ('Парацетамол', 80, 40);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapGet("/meds", async (string? q) =>
{
    return Results.Ok();
});

app.MapPost("/sale", async (SaleReq s) =>
{
    return Results.Ok();
});

app.MapGet("/meds/report", () =>
{
    return Results.Ok();
});

app.Run();
record SaleReq(int MedId, int Qty);
```

---

## Билет 9 — «Университет: студенты и оценки»

**Задание:** Эндпоинты:
- `GET /students` — `ExecuteReader` (соединение + команда + async + пул);
- `GET /avg-grade?studentId=` — `ExecuteScalar` с параметром;
- `POST /grades` — `ExecuteNonQuery` в транзакции;
- `GET /grades/dapper` — Dapper;
- `GET /grades/dataset` — `DataSet`/`DataAdapter`/`DataTable`.

**`init.sql`**
```sql
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);
CREATE TABLE grades (
    id SERIAL PRIMARY KEY,
    student_id INT REFERENCES students(id),
    grade INT NOT NULL
);
INSERT INTO students (name) VALUES ('Иван'), ('Мария');
INSERT INTO grades (student_id, grade) VALUES (1,5), (1,4), (2,5);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapGet("/students", async () =>
{
    return Results.Ok();
});

app.MapGet("/avg-grade", async (int studentId) =>
{
    return Results.Ok();
});

app.MapPost("/grades", async (GradeReq g) =>
{
    return Results.Ok();
});

app.MapGet("/grades/dapper", async () =>
{
    return Results.Ok();
});

app.MapGet("/grades/dataset", () =>
{
    return Results.Ok();
});

app.Run();
record GradeReq(int StudentId, int Grade);
```

---

## Билет 10 — «Ресторан: меню и заказы»

**Задание:** Эндпоинты:
- `POST /menu/add` — добавить блюдо: `DataAdapter.Fill` → `DataTable.Rows.Add` → `InsertCommand` + `Update` (соединение + пул);
- `POST /orders` — заказ в транзакции с параметрами (команда + async);
- `GET /menu/dapper` — Dapper;
- `GET /orders/report` — `DataSet`/`DataAdapter`/`DataTable`.

**`init.sql`**
```sql
CREATE TABLE menu (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price NUMERIC(10,2) NOT NULL
);
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    menu_id INT REFERENCES menu(id),
    qty INT NOT NULL
);
INSERT INTO menu (name, price) VALUES ('Борщ', 300), ('Салат', 200);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapPost("/menu/add", (string name, decimal price) =>
{
    return Results.Ok();
});

app.MapPost("/orders", async (OrderReq o) =>
{
    return Results.Ok();
});

app.MapGet("/menu/dapper", async () =>
{
    return Results.Ok();
});

app.MapGet("/orders/report", () =>
{
    return Results.Ok();
});

app.Run();
record OrderReq(int MenuId, int Qty);
```

---

## Билет 11 — «Авиакомпания: рейсы и брони»

**Задание:** Эндпоинты:
- `POST /book` — бронь в транзакции: `ExecuteScalar` с `FOR UPDATE`, обновление через `ExecuteNonQuery` (соединение + команда + параметры + транзакция + async + пул);
- `GET /flights` — Dapper;
- `GET /flights/dataset` — `DataSet`/`DataAdapter`/`DataTable` с фильтром по свободным местам.

**`init.sql`**
```sql
CREATE TABLE flights (
    id SERIAL PRIMARY KEY,
    code VARCHAR(20) NOT NULL,
    seats_total INT NOT NULL,
    seats_taken INT NOT NULL DEFAULT 0
);
CREATE TABLE bookings (
    id SERIAL PRIMARY KEY,
    flight_id INT REFERENCES flights(id),
    passenger VARCHAR(100) NOT NULL
);
INSERT INTO flights (code, seats_total) VALUES ('SU100', 100), ('SU200', 80);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapPost("/book", async (BookReq b) =>
{
    return Results.Ok();
});

app.MapGet("/flights", async () =>
{
    return Results.Ok();
});

app.MapGet("/flights/dataset", () =>
{
    return Results.Ok();
});

app.Run();
record BookReq(int FlightId, string Passenger);
```

---

## Билет 12 — «Кинотеатр: сеансы и билеты»

**Задание:** Эндпоинты:
- `POST /tickets` — продажа нескольких билетов на сеанс в одной транзакции (соединение + команда + параметры + транзакция + async + пул);
- `GET /sessions/dapper` — Dapper;
- `GET /sessions/datatable` — заполнить `DataTable` из `NpgsqlDataReader` и отфильтровать через `Select`;
- `GET /tickets/dataset` — `DataSet`/`DataAdapter`.

**`init.sql`**
```sql
CREATE TABLE sessions (
    id SERIAL PRIMARY KEY,
    title VARCHAR(150) NOT NULL,
    seats INT NOT NULL
);
CREATE TABLE tickets (
    id SERIAL PRIMARY KEY,
    session_id INT REFERENCES sessions(id),
    seat INT NOT NULL
);
INSERT INTO sessions (title, seats) VALUES ('Фильм А', 50), ('Фильм Б', 30);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapPost("/tickets", async (TicketReq t) =>
{
    return Results.Ok();
});

app.MapGet("/sessions/dapper", async () =>
{
    return Results.Ok();
});

app.MapGet("/sessions/datatable", async () =>
{
    return Results.Ok();
});

app.MapGet("/tickets/dataset", () =>
{
    return Results.Ok();
});

app.Run();
record TicketReq(int SessionId, int[] Seats);
```

---

## Билет 13 — «Почта: посылки и трекинг»

**Задание:** Эндпоинты:
- `GET /packages/parallel` — параллельно `Task.WhenAll` получить count(посылки) и count(статусы) (соединение + команда + async + пул);
- `POST /packages/status` — смена статуса в транзакции с параметрами;
- `GET /packages/dapper` — Dapper;
- `GET /packages/report` — `DataSet`/`DataAdapter`/`DataTable`.

**`init.sql`**
```sql
CREATE TABLE packages (
    id SERIAL PRIMARY KEY,
    code VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'new'
);
CREATE TABLE statuses (
    id SERIAL PRIMARY KEY,
    name VARCHAR(20) NOT NULL
);
INSERT INTO packages (code, status) VALUES ('P1','new'), ('P2','sent'), ('P3','new');
INSERT INTO statuses (name) VALUES ('new'), ('sent'), ('delivered');
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using System.Diagnostics;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapGet("/packages/parallel", async () =>
{
    return Results.Ok();
});

app.MapPost("/packages/status", async (StatusReq s) =>
{
    return Results.Ok();
});

app.MapGet("/packages/dapper", async () =>
{
    return Results.Ok();
});

app.MapGet("/packages/report", () =>
{
    return Results.Ok();
});

app.Run();
record StatusReq(int PackageId, string Status);
```

---

## Билет 14 — «Онлайн-курсы: студенты и уроки»

**Задание:** Эндпоинты:
- `GET /pool-test` — время 5 открытий соединения подряд (демонстрация пула);
- `POST /enroll` — запись на курс в транзакции с проверкой мест (команда + параметры + async);
- `GET /courses/dapper` — Dapper;
- `GET /courses/dataset` — `DataSet`/`DataAdapter`, связать курсы и записи через `DataRelation`, использовать `DataTable`.

**`init.sql`**
```sql
CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    title VARCHAR(150) NOT NULL,
    seats INT NOT NULL
);
CREATE TABLE enrollments (
    id SERIAL PRIMARY KEY,
    course_id INT REFERENCES courses(id),
    student VARCHAR(100) NOT NULL
);
INSERT INTO courses (title, seats) VALUES ('ADO.NET', 10), ('PostgreSQL', 8);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using System.Diagnostics;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapGet("/pool-test", async () =>
{
    return Results.Ok();
});

app.MapPost("/enroll", async (EnrollReq e) =>
{
    return Results.Ok();
});

app.MapGet("/courses/dapper", async () =>
{
    return Results.Ok();
});

app.MapGet("/courses/dataset", () =>
{
    return Results.Ok();
});

app.Run();
record EnrollReq(int CourseId, string Student);
```

---

## Билет 15 — «Комплексный: заказы и склад»

**Задание:** Эндпоинты:
- `POST /order` — оформление заказа через Dapper в транзакции (`BeginTransaction`, `ExecuteScalar` с `FOR UPDATE`, `ExecuteAsync`), проверка через параметризованный запрос (соединение + параметры + транзакция + async + пул);
- `GET /inventory` — остатки через Dapper;
- `GET /orders/report` — отчёт через `DataSet`/`DataAdapter`/`DataTable`;
- `GET /pool-info` — время открытия соединений (демонстрация пула).

**`init.sql`**
```sql
CREATE TABLE inventory (
    product_id INT PRIMARY KEY,
    stock INT NOT NULL
);
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    product_id INT,
    quantity INT,
    created_at TIMESTAMP DEFAULT NOW()
);
INSERT INTO inventory VALUES (1, 100), (2, 50);
```

**`Program.cs` (каркас)**
```csharp
using System.Data;
using System.Diagnostics;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapPost("/order", async (OrderReq req) =>
{
    return Results.Ok();
});

app.MapGet("/inventory", async () =>
{
    return Results.Ok();
});

app.MapGet("/orders/report", () =>
{
    return Results.Ok();
});

app.MapGet("/pool-info", async () =>
{
    return Results.Ok();
});

app.Run();
record OrderReq(int ProductId, int Quantity);
```
