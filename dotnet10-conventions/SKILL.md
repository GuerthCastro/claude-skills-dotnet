---
name: dotnet10-conventions
description: >
  Opinionated C# and .NET 10 coding conventions built around Clean Architecture and plain
  Dapper: one type per file, file-scoped namespaces, primary constructors, explicit types
  instead of var, POCO entities with hand written SQL, versioned migration scripts, soft
  deletes, and a single LookUp catalog table. Use this skill whenever writing, reviewing,
  refactoring, or scaffolding C# code, solution structures, entities, repositories,
  services, handlers, DTOs, mappers, validators, or tests for a .NET 10 project that
  follows these conventions. Also use when creating a new project from scratch, migrating
  an existing one, or adding a new layer to an existing solution. Apply these conventions
  without exception unless the user explicitly overrides a rule for a specific case.
license: MIT
---

# .NET 10 Coding Conventions

Author: Guerth Castro (github.com/GuerthCastro). Licensed under MIT.

These are personal, opinionated conventions refined over years of production .NET work.
They are deliberately strict: the value is in consistency, not in flexibility. Adopt them
whole, or fork and change the rules you disagree with.

Everything here runs on plain Dapper and hand written SQL. There is no ORM to install and
no base library to reference: the few shared types below are small enough to paste into
your own solution and own outright.

Placeholders used throughout: `Acme` is the company or organization prefix, `Product` is
the product or bounded context name. Replace both with your own.

## Non-negotiable rules

- One file per class, interface, enum, or record. No exceptions.
- File-scoped namespaces always: `namespace Acme.Product.Domain;`. Never block-style with braces.
- Primary constructors always. Do not declare private backing fields unless the field is
  mutated after construction, consumed by reflection or serialization, or requires derivation
  with guard clauses.
- No XML doc comments (`///`). Names and tests are the documentation.
- No `var`. Always use explicit types.
- PascalCase for classes, methods, properties, fields, parameters, and locals.
- No em dashes in comments or documentation. Use commas, periods, or restructured sentences.
- Never change code that was not explicitly requested: no reformatting, no casing changes,
  no namespace style changes as a side effect of another edit.

## Solution and project structure

Clean Architecture, strictly:

```
Acme.Product.Api               # ASP.NET Core, controllers, Program.cs, middleware
Acme.Product.Application       # DTOs, service interfaces, handlers, mappers, validators
Acme.Product.Domain            # Entities, repository interfaces, domain enums
Acme.Product.Infrastructure    # Dapper repositories, connection factory, external services
Acme.Product.Migrations        # Versioned SQL scripts, embedded as resources
Acme.Product.Tests/
  Application.Tests/
  Controller.Tests/
  Data.Tests/
  Tests.Common/
```

Dependency direction: Api depends on Application, Application depends on Domain.
Infrastructure implements Domain interfaces and is wired only at the composition root.
Dapper is referenced by Infrastructure only. If `using Dapper;` appears in Application or
Domain, a boundary was crossed.

## Entities (Domain layer)

Entities are plain POCOs. No attributes, no base library, no inheritance from anything a
framework owns. The only shared type is a small base class you keep in your Domain project:

```csharp
namespace Acme.Product.Domain.Entities;

public abstract class EntityBase
{
    public long Id { get; set; }
    public Guid EntityKey { get; set; }
    public long CreatedBy { get; set; }
    public DateTime CreatedOn { get; set; }
    public long? UpdatedBy { get; set; }
    public DateTime? UpdatedOn { get; set; }
    public bool IsDeleted { get; set; }
}
```

An entity then carries nothing but its own data:

```csharp
namespace Acme.Product.Domain.Entities;

public class Order : EntityBase
{
    public long CustomerId { get; set; }
    public string Reference { get; set; } = string.Empty;
    public decimal Total { get; set; }
    public string? Notes { get; set; }
}
```

Rules that follow from that shape:

- Every table has a `long` surrogate primary key named `Id`, plus a `Guid EntityKey` for
  external exposure. Expose `EntityKey` in URLs, payloads, and logs. Keep `Id` inside the
  database boundary.
- All foreign keys are `long`, never `Guid`. Joining on a 16 byte random value is a cost you
  pay on every index page.
- Soft delete always, via `IsDeleted`. Never physically delete a row. Every read filters
  `IsDeleted = 0`.
- Nullable reference types on. A non nullable string property gets `= string.Empty` so the
  compiler is satisfied without a constructor ceremony.
- The property name matches the column name. When they cannot match, alias in the SQL rather
  than configuring a global mapper.

## Schema and migrations

Schema lives in SQL, in source control, and is applied by a runner. Do not generate it from
attributes or from code, and never let an application create or alter a table at startup
outside the runner.

- One numbered script per change, never edited after it ships: `0007_add_order_notes.sql`.
- Scripts are idempotent where the runner does not track state, and forward only. Roll
  forward with a new script rather than editing an old one.
- Applied with DbUp, Flyway, or Grate. DbUp fits a .NET solution well: scripts as embedded
  resources in `Acme.Product.Migrations`, run from a console entry point in CI.
- Every table gets the base columns:

```sql
CREATE TABLE [Order] (
    Id          BIGINT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    EntityKey   UNIQUEIDENTIFIER NOT NULL DEFAULT NEWID(),
    CustomerId  BIGINT NOT NULL,
    Reference   NVARCHAR(50) NOT NULL,
    Total       DECIMAL(18,2) NOT NULL,
    Notes       NVARCHAR(500) NULL,
    CreatedBy   BIGINT NOT NULL,
    CreatedOn   DATETIME2 NOT NULL,
    UpdatedBy   BIGINT NULL,
    UpdatedOn   DATETIME2 NULL,
    IsDeleted   BIT NOT NULL DEFAULT 0,
    CONSTRAINT FK_Order_Customer FOREIGN KEY (CustomerId) REFERENCES [Customer](Id)
);

CREATE UNIQUE INDEX UX_Order_EntityKey ON [Order](EntityKey);
CREATE INDEX IX_Order_CustomerId ON [Order](CustomerId) WHERE IsDeleted = 0;
```

Index every foreign key and every column used in a `WHERE` clause. On SQL Server, make the
soft delete filter part of the index. On PostgreSQL, the same applies with a partial index.

## Data access with Dapper

Connections come from a factory, never from a connection string read inside a repository:

```csharp
// Domain/Interfaces/IDbConnectionFactory.cs
using System.Data;

namespace Acme.Product.Domain.Interfaces;

public interface IDbConnectionFactory
{
    Task<IDbConnection> CreateAsync(CancellationToken CancellationToken);
}
```

```csharp
// Infrastructure/Data/SqlConnectionFactory.cs
using System.Data;
using Microsoft.Data.SqlClient;
using Acme.Product.Domain.Interfaces;

namespace Acme.Product.Infrastructure.Data;

public class SqlConnectionFactory(string ConnectionString) : IDbConnectionFactory
{
    public async Task<IDbConnection> CreateAsync(CancellationToken CancellationToken)
    {
        SqlConnection Connection = new(ConnectionString);
        await Connection.OpenAsync(CancellationToken);
        return Connection;
    }
}
```

A repository interface lives in Domain and speaks in entities:

```csharp
// Domain/Interfaces/IOrderRepository.cs
using Acme.Product.Domain.Entities;

namespace Acme.Product.Domain.Interfaces;

public interface IOrderRepository
{
    Task<Order?> GetByKeyAsync(Guid EntityKey, CancellationToken CancellationToken);
    Task<IReadOnlyList<Order>> GetByCustomerAsync(long CustomerId, CancellationToken CancellationToken);
    Task<long> CreateAsync(Order Order, CancellationToken CancellationToken);
    Task SoftDeleteAsync(long Id, long DeletedBy, CancellationToken CancellationToken);
}
```

The implementation lives in Infrastructure and holds the SQL:

```csharp
// Infrastructure/Repositories/OrderRepository.cs
using System.Data;
using Dapper;
using Acme.Product.Domain.Entities;
using Acme.Product.Domain.Interfaces;

namespace Acme.Product.Infrastructure.Repositories;

public class OrderRepository(IDbConnectionFactory ConnectionFactory) : IOrderRepository
{
    private const string SelectColumns = """
        Id, EntityKey, CustomerId, Reference, Total, Notes,
        CreatedBy, CreatedOn, UpdatedBy, UpdatedOn, IsDeleted
        """;

    public async Task<Order?> GetByKeyAsync(Guid EntityKey, CancellationToken CancellationToken)
    {
        string Sql = $"""
            SELECT {SelectColumns}
            FROM [Order]
            WHERE EntityKey = @EntityKey AND IsDeleted = 0
            """;

        using IDbConnection Connection = await ConnectionFactory.CreateAsync(CancellationToken);
        CommandDefinition Command = new(Sql, new { EntityKey }, cancellationToken: CancellationToken);
        return await Connection.QuerySingleOrDefaultAsync<Order>(Command);
    }

    public async Task<IReadOnlyList<Order>> GetByCustomerAsync(long CustomerId, CancellationToken CancellationToken)
    {
        string Sql = $"""
            SELECT {SelectColumns}
            FROM [Order]
            WHERE CustomerId = @CustomerId AND IsDeleted = 0
            ORDER BY CreatedOn DESC
            """;

        using IDbConnection Connection = await ConnectionFactory.CreateAsync(CancellationToken);
        CommandDefinition Command = new(Sql, new { CustomerId }, cancellationToken: CancellationToken);
        IEnumerable<Order> Results = await Connection.QueryAsync<Order>(Command);
        return Results.ToList();
    }

    public async Task<long> CreateAsync(Order Order, CancellationToken CancellationToken)
    {
        string Sql = """
            INSERT INTO [Order] (EntityKey, CustomerId, Reference, Total, Notes, CreatedBy, CreatedOn, IsDeleted)
            OUTPUT INSERTED.Id
            VALUES (@EntityKey, @CustomerId, @Reference, @Total, @Notes, @CreatedBy, @CreatedOn, 0)
            """;

        using IDbConnection Connection = await ConnectionFactory.CreateAsync(CancellationToken);
        CommandDefinition Command = new(Sql, Order, cancellationToken: CancellationToken);
        return await Connection.ExecuteScalarAsync<long>(Command);
    }

    public async Task SoftDeleteAsync(long Id, long DeletedBy, CancellationToken CancellationToken)
    {
        string Sql = """
            UPDATE [Order]
            SET IsDeleted = 1, UpdatedBy = @DeletedBy, UpdatedOn = SYSUTCDATETIME()
            WHERE Id = @Id AND IsDeleted = 0
            """;

        using IDbConnection Connection = await ConnectionFactory.CreateAsync(CancellationToken);
        CommandDefinition Command = new(Sql, new { Id, DeletedBy }, cancellationToken: CancellationToken);
        await Connection.ExecuteAsync(Command);
    }
}
```

Dapper rules:

- Parameters always. Never interpolate a value into SQL. The only interpolation allowed is a
  compile time constant such as the `SelectColumns` list above, which never contains input.
- Raw string literals (`"""`) for every query longer than one line. No `@"..."` and no string
  concatenation across lines.
- `CommandDefinition` so the `CancellationToken` reaches the driver. A repository method
  without a token is not finished.
- `QuerySingleOrDefaultAsync` when zero or one row is expected. `QueryFirstOrDefaultAsync`
  only when the query is deliberately ordered and truncated. `QuerySingleAsync` when zero rows
  is a bug and you want the exception.
- Materialize before returning. `QueryAsync` is buffered by default, so return
  `IReadOnlyList<T>`, not the raw `IEnumerable<T>`, and never leak a live reader past the
  connection scope.
- No Dapper.Contrib, no Dapper.SimpleCRUD, no query builders. Hand written SQL is the point.
- No stored procedures for CRUD. Reserve them for set based work that genuinely belongs in the
  database.
- Repositories return domain entities, never DTOs. Mapping to DTOs happens in Application.

### Multiple result sets

Prefer one round trip over several:

```csharp
string Sql = """
    SELECT Id, EntityKey, Reference, Total FROM [Order] WHERE Id = @Id AND IsDeleted = 0;
    SELECT Id, OrderId, ProductId, Quantity FROM [OrderLine] WHERE OrderId = @Id AND IsDeleted = 0;
    """;

using IDbConnection Connection = await ConnectionFactory.CreateAsync(CancellationToken);
CommandDefinition Command = new(Sql, new { Id }, cancellationToken: CancellationToken);
using SqlMapper.GridReader Reader = await Connection.QueryMultipleAsync(Command);

Order? Order = await Reader.ReadSingleOrDefaultAsync<Order>();
IEnumerable<OrderLine> Lines = await Reader.ReadAsync<OrderLine>();
```

### Joins and multi mapping

Use `splitOn` and keep the split column immediately after the last column of the previous
entity:

```csharp
string Sql = """
    SELECT o.Id, o.Reference, o.Total, c.Id, c.Name, c.Email
    FROM [Order] o
    INNER JOIN [Customer] c ON c.Id = o.CustomerId
    WHERE o.IsDeleted = 0 AND c.IsDeleted = 0
    """;

IEnumerable<Order> Results = await Connection.QueryAsync<Order, Customer, Order>(
    Sql,
    (Order, Customer) =>
    {
        Order.Customer = Customer;
        return Order;
    },
    splitOn: "Id");
```

### Pagination

Keyset pagination when the list is ordered and large, `OFFSET FETCH` when the caller needs
arbitrary page numbers. Never fetch everything and page in memory.

```sql
SELECT Id, EntityKey, Reference, Total
FROM [Order]
WHERE IsDeleted = 0
ORDER BY CreatedOn DESC, Id DESC
OFFSET @Skip ROWS FETCH NEXT @Take ROWS ONLY;
```

### Transactions

Open one connection, one transaction, and pass both down. Do not nest connection factories
inside a unit of work:

```csharp
using IDbConnection Connection = await ConnectionFactory.CreateAsync(CancellationToken);
using IDbTransaction Transaction = Connection.BeginTransaction();

try
{
    await Connection.ExecuteAsync(new CommandDefinition(InsertOrderSql, Order, Transaction, cancellationToken: CancellationToken));
    await Connection.ExecuteAsync(new CommandDefinition(InsertLinesSql, Lines, Transaction, cancellationToken: CancellationToken));
    Transaction.Commit();
}
catch
{
    Transaction.Rollback();
    throw;
}
```

When a use case spans several repositories, the transaction belongs in an application level
unit of work that hands the same connection and transaction to each repository. It does not
belong inside a single repository method.

### PostgreSQL differences

The conventions are provider agnostic, with three adjustments:

- `NpgsqlConnection` in the factory, `bigserial` for the identity column, `uuid` for the key.
- Quote identifiers only when the schema uses mixed case. Prefer lower case table and column
  names, then set `DefaultTypeMap.MatchNamesWithUnderscores = true` once at startup if the
  schema uses snake case.
- `INSERT ... RETURNING Id` instead of `OUTPUT INSERTED.Id`.

## LookUp pattern for catalogs

Every catalog or reference list lives in one `LookUp` table, discriminated by `LookUpType`.
Do not create a table per catalog.

```sql
CREATE TABLE LookUp (
    Id          BIGINT IDENTITY(1,1) NOT NULL PRIMARY KEY,
    EntityKey   UNIQUEIDENTIFIER NOT NULL DEFAULT NEWID(),
    ParentId    BIGINT NULL,
    Name        NVARCHAR(250) NOT NULL,
    Code        NVARCHAR(250) NULL,
    Data        NVARCHAR(MAX) NULL,
    LookUpType  INT NOT NULL,
    CreatedBy   BIGINT NOT NULL,
    CreatedOn   DATETIME2 NOT NULL,
    UpdatedBy   BIGINT NULL,
    UpdatedOn   DATETIME2 NULL,
    IsDeleted   BIT NOT NULL DEFAULT 0,
    CONSTRAINT FK_LookUp_Parent FOREIGN KEY (ParentId) REFERENCES LookUp(Id)
);

CREATE INDEX IX_LookUp_Type ON LookUp(LookUpType) WHERE IsDeleted = 0;
```

```csharp
namespace Acme.Product.Domain.Entities;

public class LookUp : EntityBase
{
    public long? ParentId { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Code { get; set; } = string.Empty;
    public string Data { get; set; } = string.Empty;
    public int LookUpType { get; set; }
}
```

Catalog types are an enum in Infrastructure:

```csharp
namespace Acme.Product.Infrastructure.Enums;

public enum LookUpType
{
    OrderStatus = 1,
    PaymentMethod = 2
}
```

Each catalog type still gets its own DTO in Application and its own mapper, so consumers
never see the generic `LookUp` shape:

```csharp
// Application/Dtos/OrderStatus.cs
namespace Acme.Product.Application.Dtos;

public class OrderStatus
{
    public long Id { get; set; }
    public string Label { get; set; } = string.Empty;
    public string Code { get; set; } = string.Empty;
}
```

```csharp
// Infrastructure/Mappers/OrderStatusMapper.cs
using AutoMapper;
using Acme.Product.Application.Dtos;
using Acme.Product.Domain.Entities;
using Acme.Product.Infrastructure.Enums;

namespace Acme.Product.Infrastructure.Mappers;

public static class OrderStatusMapper
{
    private static readonly MapperConfiguration _fromLookUpConfig = new(cfg => cfg
        .CreateMap<LookUp, OrderStatus>()
        .ForMember(dest => dest.Id,    opt => opt.MapFrom(src => src.Id))
        .ForMember(dest => dest.Label, opt => opt.MapFrom(src => src.Name))
        .ForMember(dest => dest.Code,  opt => opt.MapFrom(src => src.Code))
    );

    private static readonly MapperConfiguration _toLookUpConfig = new(cfg => cfg
        .CreateMap<OrderStatus, LookUp>()
        .ForMember(dest => dest.Id,         opt => opt.MapFrom(src => src.Id))
        .ForMember(dest => dest.Name,       opt => opt.MapFrom(src => src.Label))
        .ForMember(dest => dest.Code,       opt => opt.MapFrom(src => src.Code))
        .ForMember(dest => dest.LookUpType, opt => opt.MapFrom(src => (int)LookUpType.OrderStatus))
    );

    public static OrderStatus MapOrderStatus(this LookUp LookUp)
    {
        Mapper Mapper = new(_fromLookUpConfig);
        return Mapper.Map<OrderStatus>(LookUp);
    }

    public static List<OrderStatus> MapOrderStatus(this IEnumerable<LookUp> LookUps)
    {
        Mapper Mapper = new(_fromLookUpConfig);
        return Mapper.Map<List<OrderStatus>>(LookUps);
    }

    public static LookUp Map(this OrderStatus OrderStatus)
    {
        Mapper Mapper = new(_toLookUpConfig);
        return Mapper.Map<LookUp>(OrderStatus);
    }
}
```

## Mapping rules

- Pin AutoMapper to 13.0.1, the last version under the MIT license. Later versions changed
  licensing terms, so an upgrade is a legal decision, not a routine bump.
- Static mapper pattern: `private static readonly MapperConfiguration`.
- Every `ForMember` is explicit. Never rely on convention based mapping.
- One mapper file per entity and DTO pair.
- Hand written extension methods are an acceptable substitute for AutoMapper. Reflection based
  mapping in a hot path is not.

## Application layer

- DTOs in `Application/Dtos/`, one per file.
- Service interfaces in `Application/Interfaces/`, one per file.
- Handlers and use cases in `Application/Handlers/`, one per file.
- Validation with FluentValidation, one validator per DTO or command.

## Api layer

- Versioned routes: `/api/v1/`.
- JWT bearer authentication in every environment, including local development. Never leave an
  endpoint anonymous because it is "only dev".
- Controllers are thin: model binding, authorization, delegation to a handler, status code.
- Route parameters carry `EntityKey`, never `Id`.

## Tests

- MSTest as the default framework.
- Moq for mocking, Bogus for test data, AwesomeAssertions for assertions.
- One test file per class under test.
- Class naming: `{ClassName}Tests`.
- Method naming: `{MethodName}_Should{ExpectedBehavior}_When{Condition}`.
- Repository tests run against a real database, not a mocked `IDbConnection`. Hand written SQL
  is exactly the part a mock cannot verify. Use a container or a disposable local database, and
  apply the migration scripts as part of the fixture.

## Solution file

- Prefer the `.slnx` format on Visual Studio 2026 and current SDKs.
- Solution folder `Name` attributes start and end with a slash: `Name="/01 - Application/"`.
- Standard folders: `01 - Application`, `02 - Documentation`, `03 - Workflows`, `04 - Tests`.

## What not to do

- No `var`.
- No Entity Framework, no LINQ to SQL, no lazy loading.
- No Dapper.Contrib, Dapper.SimpleCRUD, or query builder libraries.
- No SQL string interpolation with runtime values. Parameters, always.
- No `using Dapper;` outside Infrastructure.
- No block-style namespaces.
- No XML doc comments.
- No separate catalog tables. Use the LookUp pattern.
- No physical deletes. Soft delete via `IsDeleted`.
- No `Guid` foreign keys. Use `long` internally and expose `EntityKey`.
- No schema generated from code at startup. Versioned scripts, applied by a runner.
- No em dashes in comments or documentation.
- No unrequested edits bundled into a requested change.