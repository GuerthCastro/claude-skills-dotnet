---
name: solid-review
description: >
  Audit any C#, TypeScript, or Angular code against the five SOLID principles and Clean
  Architecture layering rules, then produce a severity ranked report with concrete
  refactorings. Use this skill whenever the user asks to review code for SOLID compliance,
  identify design violations, refactor toward Clean Architecture, or get a design quality
  report on a class, service, handler, component, or module. Also use when the user asks
  "does this follow SOLID?", "is this clean?", "what is wrong with this design?", or pastes
  code and asks for architectural feedback.
license: MIT
---

# SOLID Review

Author: Guerth Castro (github.com/GuerthCastro). Licensed under MIT.

## Review process

When code is provided for review:

1. Identify the language and layer (C# domain entity, application handler, Angular service,
   React component, and so on).
2. Evaluate each of the five principles independently.
3. Produce a structured report using the format below.
4. Rank violations by severity: Critical (breaks architecture) over Major (degrades
   maintainability) over Minor (style preference).
5. Provide a concrete refactored version for every Critical or Major violation.

Do not pad the report. If a principle is genuinely satisfied, say PASS and move on.

## The five principles

### S: Single Responsibility Principle

One class, one reason to change.

Violations to look for:

- A class does both data access and business logic.
- A service sends emails, validates data, and writes to the database.
- A component fetches data, formats it, and manages UI state.
- A handler orchestrates several unrelated operations.
- God classes with ten or more methods spanning multiple domains.

**VIOLATION.** `OrderService` validates, persists, and notifies. The notification call is the one
that does not belong.

```csharp
public class OrderService(IOrderRepository repository, IEmailService email)
{
    public async Task<bool> AddItem(long orderId, long productId)
    {
        Order order = await repository.GetById(orderId);
        if (order.ItemCount >= order.MaxItems)
        {
            return false;
        }

        await repository.AddItem(orderId, productId);

        await email.SendOrderUpdated(orderId);
        return true;
    }
}
```

Fix: extract a notification service and let the handler coordinate the two calls.

### O: Open/Closed Principle

Open for extension, closed for modification.

Violations to look for:

- `if/else if` chains or `switch` blocks on type strings or enums that grow with every feature.
- Adding a new report type requires editing the existing report service.
- Adding a new payment method requires editing the checkout handler.
- Concrete dependencies where an abstraction belongs.

**VIOLATION.** A new export format means editing this method.

```csharp
public byte[] ExportInvoice(string format, IEnumerable<InvoiceLine> lines)
{
    if (format == "pdf")
    {
        return GeneratePdf(lines);
    }

    if (format == "excel")
    {
        return GenerateExcel(lines);
    }

    throw new NotSupportedException(format);
}
```

Fix: an `IInvoiceExporter` abstraction with `PdfInvoiceExporter` and `ExcelInvoiceExporter`
implementations, resolved by key from the container.

### L: Liskov Substitution Principle

Subtypes must be substitutable for their base types.

Violations to look for:

- A derived class throws `NotImplementedException` for an inherited member.
- An override changes expected behavior: a silent no-op, a different return shape, a new
  exception type.
- An interface declares members that some implementations cannot meaningfully implement.
- Casting to a concrete type to reach behavior: `if (repository is SqlOrderRepository sql)`.

### I: Interface Segregation Principle

No class should be forced to implement members it does not use.

Violations to look for:

- Fat interfaces with fifteen or more members where implementations use three.
- A generic `IRepository<T>` forcing `GetAll`, `GetById`, `Create`, `Update`, `Delete`,
  `Paginate`, `Search`, and `BulkInsert` onto every entity.
- A service interface mixing reads and writes when consumers only need one side.

**VIOLATION.** A read only reporting consumer is forced to see Create, Update, and Delete.

```csharp
public interface IOrderRepository
{
    Task<Order> GetById(long id);
    Task<IEnumerable<Order>> GetAll();
    Task Create(Order order);
    Task Update(Order order);
    Task Delete(long id);
}
```

Fix: split into `IOrderReader` and `IOrderWriter`. Compose them where both are needed.

### D: Dependency Inversion Principle

Depend on abstractions, not concretions.

Violations to look for:

- `new` instantiating a service or repository inside business logic.
- A direct dependency on `SqlConnection` or `NpgsqlConnection` inside a service.
- The Application layer importing Infrastructure types.
- Static service classes holding mutable state.

**VIOLATION.** The handler builds its own concrete repository and connection.

```csharp
public class GetOrderHandler
{
    public async Task<OrderDto> Handle(long id)
    {
        OrderRepository repository = new(new SqlConnection("..."));
        Order order = await repository.GetById(id);
        return order.MapToDto();
    }
}
```

Fix: inject `IOrderRepository` through the primary constructor and register it in the
container at the composition root.

## Report format

Produce this structure for every review:

```
## SOLID Review: [ClassName or filename]
**Language/Layer:** C# Application Handler, Angular Service, and so on

### S: Single Responsibility
Status: PASS / VIOLATION (Critical, Major, Minor)
Finding: [what the class does beyond its single responsibility]
Fix: [concrete action]

### O: Open/Closed
Status: PASS / VIOLATION
Finding: ...
Fix: ...

### L: Liskov Substitution
Status: PASS / VIOLATION / N/A (no inheritance)
Finding: ...

### I: Interface Segregation
Status: PASS / VIOLATION / N/A (no interfaces)
Finding: ...

### D: Dependency Inversion
Status: PASS / VIOLATION
Finding: ...
Fix: ...

### Priority violations
1. [Critical] DIP: handler instantiates a concrete repository
2. [Major] SRP: service also sends the notification email

### Refactored version
[full corrected code block]
```

## Clean Architecture layering violations

Beyond SOLID, flag these cross cutting problems:

- A Domain entity imports Application or Infrastructure types (dependency direction violation).
- A controller contains business logic that belongs in a handler.
- A repository returns DTOs instead of domain entities. Mapping belongs in Application.
- Infrastructure calls an external API directly instead of implementing an interface declared
  in Application or Domain.
- A handler touches `HttpContext`, leaking an infrastructure concern into the application layer.
- A DI registration in the wrong assembly, so a layer has to reference downward to compile.