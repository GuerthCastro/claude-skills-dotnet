---
name: dotnet10-conventions
description: >
  Opinionated C# and .NET 10 coding conventions built around Clean Architecture: one type
  per file, file-scoped namespaces, primary constructors, explicit types instead of var,
  Dapper for data access, attribute-driven entities, soft deletes, and a single LookUp
  catalog table. Use this skill whenever writing, reviewing, refactoring, or scaffolding
  C# code, solution structures, entities, repositories, services, handlers, DTOs, mappers,
  validators, or tests for a .NET 10 project that follows these conventions. Also use when
  creating a new project from scratch, migrating an existing one, or adding a new layer to
  an existing solution. Apply these conventions without exception unless the user
  explicitly overrides a rule for a specific case.
license: MIT
---

# .NET 10 Coding Conventions

Author: Guerth Castro (github.com/GuerthCastro). Licensed under MIT.

These are personal, opinionated conventions refined over years of production .NET work.
They are deliberately strict: the value is in consistency, not in flexibility. Adopt them
whole, or fork and change the rules you disagree with.

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
Acme.Product.Infrastructure    # Repository and external service implementations, DB bootstrapper
Acme.Product.Tests/
  Application.Tests/
  Controller.Tests/
  Data.Tests/
  Tests.Common/
```

Dependency direction: Api depends on Application, Application depends on Domain.
Infrastructure implements Domain interfaces and is wired only at the composition root.

## Entities (Domain layer)

These samples assume a shared data library that provides an `EntityBase` type plus `Table`,
`Column`, and validation attributes, and that generates schema from those attributes. Swap in
the equivalent from your own base library if the names differ.

```csharp
using Core.Data.Attributes;
using Core.Data.Entities;

namespace Acme.Product.Domain.Entities;

[Table(Name = "TableName", CreateOrder = 1)]
public class MyEntity : EntityBase
{
    [RequiredField]
    [Column(Name = "ColumnName", IsNullable = false)]
    public string Name { get; set; } = string.Empty;

    [Column(Name = "RelatedId", IsNullable = false, ForeignKey = "RelatedTable(Id)")]
    public long RelatedId { get; set; }

    [Column(Name = "OptionalField", IsNullable = true)]
    public string? OptionalField { get; set; }
}
```

`EntityBase` provides `Id` (long, primary key, auto increment), `EntityKey` (Guid),
`CreatedBy`, `UpdatedBy` (long), `CreatedOn`, `UpdatedOn` (DateTime) and `IsDeleted` (bool).
Never redeclare these on a derived entity.

Rules that follow from that shape:

- All foreign keys are `long`, never `Guid`. Expose `EntityKey` externally, keep `Id` internal.
- `CreateOrder` must respect foreign key dependencies: referenced tables get lower numbers.
- Soft delete always, via `IsDeleted`. Never physically delete a row.

## LookUp pattern for catalogs

Every catalog or reference list lives in one `LookUp` table, discriminated by `LookUpType`.
Do not create a table per catalog.

```csharp
[Table(Name = "LookUp", CreateOrder = 1)]
public class LookUp : EntityBase
{
    [Column(Name = "ParentId", IsNullable = true)]
    public long? ParentId { get; set; }

    [RequiredField, StringLength(250)]
    [Column(Name = "Name", IsNullable = false)]
    public string Name { get; set; } = string.Empty;

    [RequiredField, StringLength(250)]
    [Column(Name = "Code", IsNullable = true)]
    public string Code { get; set; } = string.Empty;

    [RequiredField]
    [Column(Name = "Data", IsNullable = true)]
    public string Data { get; set; } = string.Empty;

    [Column(Name = "LookUpType", IsNullable = false)]
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
using Core.Data.Entities;
using Acme.Product.Application.Dtos;
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

    public static OrderStatus MapOrderStatus(this LookUp lookUp)
    {
        Mapper mapper = new(_fromLookUpConfig);
        return mapper.Map<OrderStatus>(lookUp);
    }

    public static List<OrderStatus> MapOrderStatus(this IEnumerable<LookUp> lookUps)
    {
        Mapper mapper = new(_fromLookUpConfig);
        return mapper.Map<List<OrderStatus>>(lookUps);
    }

    public static LookUp Map(this OrderStatus orderStatus)
    {
        Mapper mapper = new(_toLookUpConfig);
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

## Data access

- Dapper only. No Entity Framework, no LINQ to SQL, no lazy loading.
- Repository interfaces in Domain, implementations in Infrastructure.
- Repositories return domain entities, never DTOs. Mapping belongs in Application.
- Schema comes from entity attributes, not from hand written migration scripts.

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

## Tests

- MSTest as the default framework.
- Moq for mocking, Bogus for test data, AwesomeAssertions for assertions.
- One test file per class under test.
- Class naming: `{ClassName}Tests`.
- Method naming: `{MethodName}_Should{ExpectedBehavior}_When{Condition}`.

## Solution file

- Prefer the `.slnx` format on Visual Studio 2026 and current SDKs.
- Solution folder `Name` attributes start and end with a slash: `Name="/01 - Application/"`.
- Standard folders: `01 - Application`, `02 - Documentation`, `03 - Workflows`, `04 - Tests`.

## What not to do

- No `var`.
- No Entity Framework.
- No block-style namespaces.
- No XML doc comments.
- No separate catalog tables. Use the LookUp pattern.
- No physical deletes. Soft delete via `IsDeleted`.
- No `Guid` foreign keys. Use `long` internally and expose `EntityKey`.
- No em dashes in comments or documentation.
- No unrequested edits bundled into a requested change.
