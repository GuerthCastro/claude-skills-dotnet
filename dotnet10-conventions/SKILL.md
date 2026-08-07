---
name: dotnet10-conventions
description: >
  Opinionated C# and .NET 10 coding conventions built around Clean Architecture and plain
  Dapper: one type per file, file-scoped namespaces, primary constructors, camelCase locals
  and fields with no underscore prefix, POCO entities with hand written SQL, versioned
  migration scripts, soft deletes, and a single LookUp catalog table. Use this skill
  whenever writing, reviewing,
  refactoring, or scaffolding C# code, solution structures, entities, repositories,
  services, handlers, DTOs, mappers, or validators for a .NET 10 project that
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
- No XML doc comments (`///`) and no inline comments. Names and tests are the documentation. When a
  comment feels necessary, the code is not clear enough yet: extract a named method, rename the
  variable, or tighten the type so the fact becomes structural. Reasoning that genuinely cannot live
  in code goes in the commit message or the pull request, not the source. There is exactly one
  standing exception: `// Arrange`, `// Act`, and `// Assert` in a test body, which mark structure
  rather than explain code. Anything else is granted per case, never by default, and when granted
  the comment is written in English, kept to a few lines, and carries no em dashes.
- `var` by default. Reach for an explicit type when the declared type carries weight: when it is
  not the type on the right hand side, when it keeps a cast out of the code that follows, or when
  the right hand side does not make the type obvious. `using IDbConnection connection = new
  SqlConnection(connectionString);` is explicit because the interface is the point.
- PascalCase for classes, interfaces, methods, and properties. camelCase for private fields,
  parameters, and locals. Never prefix a private field with an underscore.
- Constants follow visibility, not constness. A `private const` is an implementation detail like any
  other private field, so it is camelCase. A public or internal constant is API surface like a
  property, so it is PascalCase, and a class that exists to hold a catalog of constants is public by
  definition and lands there on its own.
- No noise affixes. A method that returns a `Task` is not called `SendAsync`, it is called `Send`,
  and the signature already says the rest. The same goes for a `Sync` counterpart or a prefix that
  restates the type. Affixes that name what a type is stay: the `I` on an interface, and the
  `Controller`, `Repository`, `Options`, `Model`, and `ViewModel` that place a type in the
  architecture. Framework members keep the names their authors gave them: `OpenAsync`, `QueryAsync`,
  and `RenderSectionAsync` are not yours to rename.
- One statement per line. A call, a fluent chain, a method signature, or a concatenation that fits
  on one line stays on one line, however long it runs. Object and collection initializers and
  constructor declarations are exempt: their braces stay open across lines, one member assignment
  per line, however few members there are:

```csharp
var order = new Order
{
    CustomerId = customerId,
    Reference = reference,
    Total = total
};
```

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

`Domain/Interfaces/IDbConnectionFactory.cs`

```csharp
using System.Data;

namespace Acme.Product.Domain.Interfaces;

public interface IDbConnectionFactory
{
    Task<IDbConnection> Create(CancellationToken cancellationToken);
}
```

`Infrastructure/Data/SqlConnectionFactory.cs`

```csharp
using System.Data;
using Microsoft.Data.SqlClient;
using Acme.Product.Domain.Interfaces;

namespace Acme.Product.Infrastructure.Data;

public class SqlConnectionFactory(string connectionString) : IDbConnectionFactory
{
    public async Task<IDbConnection> Create(CancellationToken cancellationToken)
    {
        SqlConnection connection = new(connectionString);
        await connection.OpenAsync(cancellationToken);
        return connection;
    }
}
```

A repository interface lives in Domain and speaks in entities:

`Domain/Interfaces/IOrderRepository.cs`

```csharp
using Acme.Product.Domain.Entities;

namespace Acme.Product.Domain.Interfaces;

public interface IOrderRepository
{
    Task<Order?> GetByKey(Guid entityKey, CancellationToken cancellationToken);
    Task<IReadOnlyList<Order>> GetByCustomer(long customerId, CancellationToken cancellationToken);
    Task<long> Create(Order order, CancellationToken cancellationToken);
    Task SoftDelete(long id, long deletedBy, CancellationToken cancellationToken);
}
```

The implementation lives in Infrastructure and holds the SQL:

`Infrastructure/Repositories/OrderRepository.cs`

```csharp
using System.Data;
using Dapper;
using Acme.Product.Domain.Entities;
using Acme.Product.Domain.Interfaces;

namespace Acme.Product.Infrastructure.Repositories;

public class OrderRepository(IDbConnectionFactory connectionFactory) : IOrderRepository
{
    private const string selectColumns = """
        Id, EntityKey, CustomerId, Reference, Total, Notes,
        CreatedBy, CreatedOn, UpdatedBy, UpdatedOn, IsDeleted
        """;

    public async Task<Order?> GetByKey(Guid entityKey, CancellationToken cancellationToken)
    {
        string sql = $"""
            SELECT {selectColumns}
            FROM [Order]
            WHERE EntityKey = @EntityKey AND IsDeleted = 0
            """;

        using IDbConnection connection = await connectionFactory.Create(cancellationToken);
        CommandDefinition command = new(sql, new { EntityKey = entityKey }, cancellationToken: cancellationToken);
        return await connection.QuerySingleOrDefaultAsync<Order>(command);
    }

    public async Task<IReadOnlyList<Order>> GetByCustomer(long customerId, CancellationToken cancellationToken)
    {
        string sql = $"""
            SELECT {selectColumns}
            FROM [Order]
            WHERE CustomerId = @CustomerId AND IsDeleted = 0
            ORDER BY CreatedOn DESC
            """;

        using IDbConnection connection = await connectionFactory.Create(cancellationToken);
        CommandDefinition command = new(sql, new { CustomerId = customerId }, cancellationToken: cancellationToken);
        IEnumerable<Order> results = await connection.QueryAsync<Order>(command);
        return results.ToList();
    }

    public async Task<long> Create(Order order, CancellationToken cancellationToken)
    {
        string sql = """
            INSERT INTO [Order] (EntityKey, CustomerId, Reference, Total, Notes, CreatedBy, CreatedOn, IsDeleted)
            OUTPUT INSERTED.Id
            VALUES (@EntityKey, @CustomerId, @Reference, @Total, @Notes, @CreatedBy, @CreatedOn, 0)
            """;

        using IDbConnection connection = await connectionFactory.Create(cancellationToken);
        CommandDefinition command = new(sql, order, cancellationToken: cancellationToken);
        return await connection.ExecuteScalarAsync<long>(command);
    }

    public async Task SoftDelete(long id, long deletedBy, CancellationToken cancellationToken)
    {
        string sql = """
            UPDATE [Order]
            SET IsDeleted = 1, UpdatedBy = @DeletedBy, UpdatedOn = SYSUTCDATETIME()
            WHERE Id = @Id AND IsDeleted = 0
            """;

        using IDbConnection connection = await connectionFactory.Create(cancellationToken);
        CommandDefinition command = new(sql, new { Id = id, DeletedBy = deletedBy }, cancellationToken: cancellationToken);
        await connection.ExecuteAsync(command);
    }
}
```

Dapper rules:

- Parameters always. Never interpolate a value into SQL. The only interpolation allowed is a
  compile time constant such as the `selectColumns` list above, which never contains input.
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
- No stored procedures for CRUD. Reserve them for set based work that genuinely belongs in the
  database.
- Repositories return domain entities, never DTOs. Mapping to DTOs happens in Application.

### Multiple result sets

Prefer one round trip over several:

```csharp
string sql = """
    SELECT Id, EntityKey, Reference, Total FROM [Order] WHERE Id = @Id AND IsDeleted = 0;
    SELECT Id, OrderId, ProductId, Quantity FROM [OrderLine] WHERE OrderId = @Id AND IsDeleted = 0;
    """;

using IDbConnection connection = await connectionFactory.Create(cancellationToken);
CommandDefinition command = new(sql, new { Id = id }, cancellationToken: cancellationToken);
using SqlMapper.GridReader reader = await connection.QueryMultipleAsync(command);

Order? order = await reader.ReadSingleOrDefaultAsync<Order>();
IEnumerable<OrderLine> lines = await reader.ReadAsync<OrderLine>();
```

### Joins and multi mapping

Use `splitOn` and keep the split column immediately after the last column of the previous
entity:

```csharp
string sql = """
    SELECT o.Id, o.Reference, o.Total, c.Id, c.Name, c.Email
    FROM [Order] o
    INNER JOIN [Customer] c ON c.Id = o.CustomerId
    WHERE o.IsDeleted = 0 AND c.IsDeleted = 0
    """;

IEnumerable<Order> results = await connection.QueryAsync<Order, Customer, Order>(
    sql,
    (order, customer) =>
    {
        order.Customer = customer;
        return order;
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
using IDbConnection connection = await connectionFactory.Create(cancellationToken);
using IDbTransaction transaction = connection.BeginTransaction();

try
{
    await connection.ExecuteAsync(new CommandDefinition(insertOrderSql, order, transaction, cancellationToken: cancellationToken));
    await connection.ExecuteAsync(new CommandDefinition(insertLinesSql, lines, transaction, cancellationToken: cancellationToken));
    transaction.Commit();
}
catch
{
    transaction.Rollback();
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

`Application/Dtos/OrderStatus.cs`

```csharp
namespace Acme.Product.Application.Dtos;

public class OrderStatus
{
    public long Id { get; set; }
    public string Label { get; set; } = string.Empty;
    public string Code { get; set; } = string.Empty;
}
```

`Infrastructure/Mappers/OrderStatusMapper.cs`

```csharp
using AutoMapper;
using Acme.Product.Application.Dtos;
using Acme.Product.Domain.Entities;
using Acme.Product.Infrastructure.Enums;

namespace Acme.Product.Infrastructure.Mappers;

public static class OrderStatusMapper
{
    private static readonly MapperConfiguration fromLookUpConfig = new(cfg => cfg
        .CreateMap<LookUp, OrderStatus>()
        .ForMember(dest => dest.Id,    opt => opt.MapFrom(src => src.Id))
        .ForMember(dest => dest.Label, opt => opt.MapFrom(src => src.Name))
        .ForMember(dest => dest.Code,  opt => opt.MapFrom(src => src.Code))
    );

    private static readonly MapperConfiguration toLookUpConfig = new(cfg => cfg
        .CreateMap<OrderStatus, LookUp>()
        .ForMember(dest => dest.Id,         opt => opt.MapFrom(src => src.Id))
        .ForMember(dest => dest.Name,       opt => opt.MapFrom(src => src.Label))
        .ForMember(dest => dest.Code,       opt => opt.MapFrom(src => src.Code))
        .ForMember(dest => dest.LookUpType, opt => opt.MapFrom(src => (int)LookUpType.OrderStatus))
    );

    public static OrderStatus MapOrderStatus(this LookUp lookUp)
    {
        Mapper mapper = new(fromLookUpConfig);
        return mapper.Map<OrderStatus>(lookUp);
    }

    public static List<OrderStatus> MapOrderStatus(this IEnumerable<LookUp> lookUps)
    {
        Mapper mapper = new(fromLookUpConfig);
        return mapper.Map<List<OrderStatus>>(lookUps);
    }

    public static LookUp Map(this OrderStatus orderStatus)
    {
        Mapper mapper = new(toLookUpConfig);
        return mapper.Map<LookUp>(orderStatus);
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

## Solution file

- Prefer the `.slnx` format on Visual Studio 2026 and current SDKs.
- Solution folder `Name` attributes start and end with a slash: `Name="/Source/"`.
- Standard folders:

```
Source          # Api, Application, Domain, Infrastructure
Tests           # every test project
Documentation   # architecture notes, decision records, diagrams
Pipelines       # CI definitions, Dockerfiles, deployment manifests
```

- No numeric prefixes. A prefix encodes ordering into the name itself, so inserting a folder means
  renaming its neighbours, and the number carries no meaning to anyone reading a path in a stack
  trace or a build log. Four folders sort readably on their own.
- `Source`, not `Application`. The solution folder holds all four layers, and one of them is
  already named Application. A folder named after one of its own children is a reliable source
  of confusion in a large solution.
- `Pipelines`, not a provider specific name. The contents differ between GitHub Actions, Azure
  Pipelines, and GitLab CI, but the slot in the solution is the same, and renaming solution
  folders across active branches causes merge noise for no benefit.

## What not to do

- No Entity Framework.
- No SQL string interpolation with runtime values. Parameters, always.
- No `using Dapper;` outside Infrastructure.
- No block-style namespaces.
- No statement wrapped across lines when it fits on one, initializers and constructors aside.
- No XML doc comments and no inline comments, apart from `// Arrange`, `// Act`, and `// Assert`
  in a test body.
- No underscore prefix on a private field.
- No `Async` or `Sync` suffix on a method you wrote, and no prefix that restates the type.
- No separate catalog tables. Use the LookUp pattern.
- No physical deletes. Soft delete via `IsDeleted`.
- No schema generated from code at startup. Versioned scripts, applied by a runner.
- No em dashes in comments or documentation.
- No unrequested edits bundled into a requested change.