---
name: dotnet10-testing
description: >
  Opinionated unit testing conventions for .NET 10 solutions built on Clean Architecture and
  Dapper: xUnit as the runner with no parameterized tests, Moq for collaborators, Bogus fakers
  as dedicated builder classes in a shared test library, AwesomeAssertions for assertions,
  result objects instead of thrown exceptions, and repository tests that run against a real
  disposable SQLite database rather than a mocked connection. Use this skill whenever writing,
  reviewing, or restructuring tests for a .NET project: service tests, controller tests,
  repository tests, test data builders, mocking strategy, test naming, or test project layout.
  Also use when setting up a test project from scratch or deciding what deserves a test at all.
license: MIT
---

# .NET Testing Conventions

Author: Guerth Castro (github.com/GuerthCastro). Licensed under MIT.

These conventions are extracted from a production identity and access management service with
675 tests, not invented for this document. They pair with `dotnet10-conventions`, which covers
the production code these tests exercise.

Placeholders: `Acme` is the company prefix, `Product` is the product or bounded context.

## Stack

| Package | Role |
| --- | --- |
| `xunit` | The runner. `[Fact]` only. |
| `xunit.runner.visualstudio` | Test discovery in the IDE. `PrivateAssets=all`. |
| `Microsoft.NET.Test.Sdk` | Required for `dotnet test`. |
| `Moq` | Mocking interfaces. |
| `Bogus` | Test data generation, in the shared test project only. |
| `AwesomeAssertions` | Assertions. The MIT licensed fork of FluentAssertions, which went commercial at v8. |
| `coverlet.collector` | Coverage collection. `PrivateAssets=all`. |

Reference exactly these. A test project that also carries MSTest, NUnit, or a Dapper mocking
package is carrying dead weight: pick one runner and delete the rest, because two runners in one
project produce discovery ambiguity and a false impression of what the tests actually use.

## Test project layout

One test project per production layer, not per production project:

```
Acme.Product.Tests/
  Application.Tests/     -> Services in the Application layer
  Controller.Tests/      -> Controllers in the API layer
  Data.Tests/            -> Repositories in the Data layer
  Tests.Common/          -> Fakers and shared infrastructure. No runner packages here.
```

`Tests.Common` references the Domain project and Bogus, nothing else. It is a library consumed
by the three runner projects, so it must not reference xUnit or Moq. If a helper in
`Tests.Common` needs a mock, the helper belongs in the runner project instead.

Inside each test project, the folder structure mirrors production exactly:

```
Application.Tests/Services/OrderServiceTests.cs   <-> Application/Services/OrderService.cs
Data.Tests/Repositories/OrderRepositoryTests.cs   <-> Data/Repositories/OrderRepository.cs
Controller.Tests/OrderControllerTests.cs          <-> Api/Controllers/v1/OrderController.cs
```

One test class per production class, always. Class name is `{ProductionClassName}Tests`. Never
split by scenario into several classes, never cover two production classes from one test class.

## Test method anatomy

Naming: `{MethodName}_Should{ExpectedResult}_When{Scenario}`.

```
Create_ShouldReturnSuccess_WhenOrderIsValid
Create_ShouldReturnFailure_WhenReferenceAlreadyExists
GetById_ShouldReturnNull_WhenOrderDoesNotExist
Cancel_ShouldReturnFailure_WhenOrderAlreadyShipped
```

The outcome comes before the condition on purpose: scanning a failing test list, the expected
behavior is what you need first, and the scenario is the qualifier.

AAA is explicit, through `// Arrange`, `// Act`, and `// Assert` comments separated by blank
lines. Every test, without exception. The comments are not decoration: they are what makes a
twenty line test scannable in review.

```csharp
[Fact]
public async Task Create_ShouldReturnFailure_WhenReferenceAlreadyExists()
{
    // Arrange
    Order Order = _orderFaker.Generate();
    Order ExistingOrder = _orderFaker.Generate();
    ExistingOrder.Reference = Order.Reference;

    _orderRepository.Setup(x => x.Get(It.IsAny<object>())).ReturnsAsync(Result.Ok(ExistingOrder));

    // Act
    Result<Order> Result = await _service.Create(Order);

    // Assert
    Result.IsFailed.Should().BeTrue();
    Result.Errors.Should().Contain(e => e.Message == "An order with this reference already exists");

    _orderRepository.Verify(x => x.Get(It.IsAny<object>()), Times.Once);
    _orderRepository.Verify(x => x.Insert(It.IsAny<Order>()), Times.Never);
}
```

Rules that example encodes:

- Ten to twenty five lines is the normal size. Longer is acceptable when a repository test has to
  insert parent rows first.
- Multiple assertions per test are the norm. There is no one assert per test rule here. A test
  asserts the outcome and then verifies the interactions that outcome implies.
- Verifying what did **not** happen matters as much as verifying what did. `Times.Never` on the
  write path is what proves the guard clause actually guarded.
- Async all the way. The test method is `async Task` and awaits directly. No `.Result`, no
  `.Wait()`, with one narrow exception noted under repository tests.

## No parameterized tests

`[Theory]` and `[InlineData]` are not used. When a method needs coverage across several input
variants, write one `[Fact]` per variant with a name that states the variant:

```csharp
VerifyPassword_ShouldReturnFalse_WhenHashIsNull
VerifyPassword_ShouldReturnFalse_WhenHashIsEmpty
VerifyPassword_ShouldReturnFalse_WhenHashIsMalformed
```

The trade is deliberate: more lines, in exchange for a failing test whose name alone tells you
which input broke without opening the file or decoding a row index.

## Assertions and failure modeling

AwesomeAssertions exclusively. Never `Assert.Equal` or any other native xUnit assertion, even
though xUnit is the runner.

```csharp
Result.IsSuccess.Should().BeTrue();
Result.Value.Id.Should().Be(InsertedId);
Result.Value.Should().OnlyContain(x => x.CustomerId == _customerId);
UpdatedOrder.Value.UpdatedOn.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromMinutes(1));
```

Production code returns a result object rather than throwing for expected failures, so tests
assert on result state, never on exceptions. `Assert.Throws`, `Should().Throw()`, and
`Record.Exception` do not appear in this convention at all. A test that wants to catch an
exception is usually pointing at production code that should have returned a failure result.

Do not write custom assertion extensions. Stock fluent chains keep the test vocabulary the same
across every project.

## Test data

Bogus, wrapped in dedicated faker classes that live in `Tests.Common`. Never an inline
`Faker<T>` inside a test method, never an object mother.

```csharp
// Tests.Common/Interfaces/IFaker.cs
namespace Acme.Product.Tests.Common.Interfaces;

public interface IFaker<T> where T : class
{
    T Generate();
    List<T> Generate(int Count = 5);
}
```

```csharp
// Tests.Common/DataFakers/FakerBase.cs
using Bogus;
using Acme.Product.Tests.Common.Interfaces;

namespace Acme.Product.Tests.Common.DataFakers;

public abstract class FakerBase<T> : IFaker<T> where T : class
{
    internal Faker<T> _faker = null!;

    public T Generate() => _faker.Generate();
    public List<T> Generate(int Count = 5) => _faker.Generate(Count);
    public virtual List<T> Generate(int Count, int StartIndex = 0) => _faker.Generate(Count);
}
```

```csharp
// Tests.Common/DataFakers/Entities/OrderFaker.cs
using Bogus;
using Acme.Product.Domain.Entities;

namespace Acme.Product.Tests.Common.DataFakers.Entities;

public class OrderFaker : FakerBase<Order>
{
    public OrderFaker()
    {
        _faker = new Faker<Order>()
            .RuleFor(x => x.Id, f => f.Random.Long(1, 999999))
            .RuleFor(x => x.EntityKey, f => Guid.NewGuid())
            .RuleFor(x => x.Reference, f => f.Commerce.Ean13())
            .RuleFor(x => x.Total, f => f.Finance.Amount(10, 5000))
            .RuleFor(x => x.CustomerId, f => f.Random.Long(1, 9999))
            .RuleFor(x => x.CreatedOn, f => f.Date.Past());
    }
}
```

Rules:

- One faker per entity and one per DTO that tests actually construct. Fakers live in
  `DataFakers/Entities/` and `DataFakers/DTOs/`, one file each.
- Strongly typed configuration objects get fakers too, under `DataFakers/Config/`.
- Never assert on a generated value. Assert on relationships between generated values, or on
  values the test set explicitly. `Order.Total.Should().Be(482.15m)` against a faked amount is a
  test that will fail on a Tuesday for no reason.
- When a test needs an exact value, set it explicitly after generating: generate the object, then
  overwrite the one field the assertion depends on. Everything else stays random.

## Mocking policy

The policy differs by layer, on purpose.

**Services and controllers: mock everything.** Every repository, every collaborating service,
every `ILogger<T>`. Loose mocking, never `MockBehavior.Strict`. Strict mocks turn every unrelated
production change into a test failure, which trains people to stop reading failures.

```csharp
_orderRepository.Setup(x => x.Get(It.IsAny<object>())).ReturnsAsync(Result.Ok(ExistingOrder));
_orderRepository.Verify(x => x.Insert(It.IsAny<Order>()), Times.Once);
```

Callback style setups where the mock has to invoke a delegate it was handed:

```csharp
_cacheService
    .Setup(x => x.GetOrCreate(It.IsAny<string>(), It.IsAny<TimeSpan>(), It.IsAny<Func<Task<List<string>>>>()))
    .Returns<string, TimeSpan, Func<Task<List<string>>>>((Key, Ttl, Factory) => Factory());
```

**Repositories: mock nothing.** Do not mock `IDbConnection` and do not use a Dapper mocking
package. Hand written SQL is precisely the part a mock cannot verify: a mocked connection will
happily accept a query with a typo in a column name.

**Dependency free services: instantiate the real thing.** A hashing service or a formatter with
no injected dependencies gets tested directly, with the real algorithm running. Same for cheap
in memory infrastructure: use a real `MemoryCache` rather than mocking `IMemoryCache`, and mock
only the logger next to it.

No hand written stubs or fakes implementing an interface. If it is an interface with behavior,
it is a Moq mock. If it is real enough to instantiate cheaply, instantiate it.

## Repository tests against a real database

Repository tests run against a real SQLite file database, one fresh copy per test class, created
from a template database checked into the test project.

```csharp
// Tests.Common/Infrastructure/RepositoryTestBase.cs
public abstract class RepositoryTestBase<T> : IDisposable where T : class
{
    protected RepositoryBase<T> Repository { get; set; } = null!;
    protected DatabaseConnection DatabaseConnection { get; }

    protected RepositoryTestBase()
    {
        RegisterTypeHandlers();

        string TestDbPath = Path.Combine(DataFolder, $"test_{Guid.NewGuid():N}.db");
        string TemplateDbPath = Path.Combine(ProjectRoot, "Data", "template.db3");

        if (File.Exists(TemplateDbPath))
        {
            File.Copy(TemplateDbPath, TestDbPath);
        }

        DatabaseConnection = new DatabaseConnection
        {
            DatabaseType = "SQLITE",
            ConnectionString = $"Data Source={TestDbPath};"
        };
    }

    public virtual void Dispose()
    {
        GC.Collect();
        GC.WaitForPendingFinalizers();
        GC.Collect();
        GC.SuppressFinalize(this);
    }
}
```

Notes that matter when copying this pattern:

- A concrete test class inherits `RepositoryTestBase<Order>` and gets a private database it can
  write to freely. Isolation comes from the file copy, not from transactions or cleanup scripts.
- Register Dapper type handlers in the base constructor. SQLite has no native `Guid` type, so
  without a handler every `EntityKey` round trip fails in a way that looks like a mapping bug.
- The `GC` calls in `Dispose` are not cargo cult. SQLite holds the file handle until the
  connection is finalized, and the next test cannot copy over a locked file.
- Constructors cannot be async, so `.GetAwaiter().GetResult()` inside the base setup is the one
  accepted place to block on a task. Nowhere else.
- Tests that need related rows insert them in dependency order inside Arrange. That makes some
  repository tests longer than service tests, which is expected.

## Controller tests

Instantiate the controller directly with mocked services. No `WebApplicationFactory`, no in
memory host. Set `ControllerContext` by hand when the action reads from the request.

```csharp
[Fact]
public async Task Create_ShouldReturnStatusCode201_WhenSuccessful()
{
    // Arrange
    OrderRequest Request = _orderRequestFaker.Generate();
    OrderInfo CreatedOrder = _orderInfoFaker.Generate();

    _orderService.Setup(x => x.Create(It.IsAny<Order>())).ReturnsAsync(Result.Ok(CreatedOrder));

    // Act
    ActionResult<OrderInfo> Result = await _controller.Create(Request);

    // Assert
    Result.Result.Should().BeOfType<CreatedAtActionResult>();

    CreatedAtActionResult CreatedResult = (Result.Result as CreatedAtActionResult)!;
    CreatedResult.StatusCode.Should().Be(201);
    CreatedResult.ActionName.Should().Be(nameof(OrderController.GetById));
    CreatedResult.Value.Should().Be(CreatedOrder);

    _orderService.Verify(x => x.Create(It.IsAny<Order>()), Times.Once);
}
```

Assert the concrete result type, the status code, and the payload. Asserting only that the
result is not null tests nothing: every action returns something.

## Setup and dependency injection

Constructor based setup, universally. xUnit constructs a new instance of the test class for every
test, so the constructor is the per test setup and no attribute is needed.

```csharp
public OrderServiceTests()
{
    _logger = new Mock<ILogger<OrderService>>();
    _orderRepository = new Mock<IOrderRepository>();
    _customerRepository = new Mock<ICustomerRepository>();

    _service = new OrderService(_logger.Object, _orderRepository.Object, _customerRepository.Object);
}
```

- No `IClassFixture`, no collection fixtures. Shared state between tests is the thing this
  convention is buying isolation against.
- No DI container in tests. Dependencies are constructed by hand and passed to the constructor.
  If wiring the class under test by hand is painful, the constructor has too many dependencies,
  and that is a design signal worth acting on rather than hiding behind a container.
- Only repository test classes implement `IDisposable`. Mock based tests have nothing to clean up.

## What gets a test

Functional code: anything that makes a decision, transforms state, or talks to a database.
Services, repositories, and controllers are the three that always get a test class.

Declarative code is exercised indirectly and does not get its own test class by default: mappers,
DTOs, entities, and validators whose rules are simple attribute style declarations. They run
through the service tests that use them, and testing them directly mostly asserts that a mapping
library still maps.

That line moves depending on the project. A validator holding real branching logic is functional
code and belongs in the first group. Draw it where it fits your codebase.

## Coverage

Coverage is collected with coverlet and read as a diagnostic. The practical target is high line
coverage on the functional code above. Whether that number becomes a gate in your pipeline is
your call, not this skill's.

## What not to do

- No MSTest, no NUnit, no second runner alongside xUnit.
- No `[Theory]` or `[InlineData]`. One `[Fact]` per variant.
- No native xUnit assertions. AwesomeAssertions only.
- No `Assert.Throws` or exception based expectations. Assert on result objects.
- No `MockBehavior.Strict`.
- No mocked database connections in repository tests.
- No inline `Faker<T>` inside a test method.
- No assertions on randomly generated values.
- No fixed seeds hiding a test that depends on specific data.
- No hand written stubs where a mock or the real object will do.
- No DI container, no `WebApplicationFactory`, no shared fixtures.
- No `.Result` or `.Wait()` outside a test base constructor.
- No test class covering more than one production class.