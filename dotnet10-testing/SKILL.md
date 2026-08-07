---
name: dotnet10-testing
description: >
  Opinionated unit testing conventions for .NET 10 solutions built on Clean Architecture and
  Dapper: xUnit as the runner with `[Theory]` reserved for variants that differ only in input
  values, Moq for collaborators, Bogus fakers as dedicated builder classes in a shared test
  library, AwesomeAssertions for assertions, result objects instead of thrown exceptions, and
  repository tests that run against the real engine in a disposable container rather than a
  mocked connection. Use this skill whenever writing,
  reviewing, or restructuring tests for a .NET project: service tests, controller tests,
  repository tests, test data builders, mocking strategy, test naming, or test project layout.
  Also use when setting up a test project from scratch or deciding what deserves a test at all.
license: MIT
---

# .NET Testing Conventions

Author: Guerth Castro (github.com/GuerthCastro). Licensed under MIT.

These conventions are extracted from a production suite of more than 600 tests, not invented for
this document. They pair with `dotnet10-conventions`, which covers the production code these
tests exercise.

Placeholders: `Acme` is the company prefix, `Product` is the product or bounded context.

## Stack

| Package | Role |
| --- | --- |
| `xunit` | The runner. `[Fact]` by default, `[Theory]` for the cases described below. |
| `xunit.runner.visualstudio` | Test discovery in the IDE. `PrivateAssets=all`. |
| `Microsoft.NET.Test.Sdk` | Required for `dotnet test`. |
| `Moq` | Mocking interfaces. |
| `Bogus` | Test data generation, in the shared test project only. |
| `AwesomeAssertions` | Assertions. The MIT licensed fork of FluentAssertions, which went commercial at v8. |
| `coverlet.collector` | Coverage collection. `PrivateAssets=all`. |

Reference exactly these. A test project that also carries MSTest, NUnit, or a Dapper mocking
package is carrying dead weight: pick one runner and delete the rest, because two runners in one
project produce discovery ambiguity and a false impression of what the tests actually use.

## Style inside test projects

Test code deviates from the production conventions in one specific way, and it is not about types
or casing: `// Arrange`, `// Act`, and `// Assert` are the only comments allowed anywhere in the
solution, and they live here. They mark structure rather than explain code, which is why they
survive a rule that otherwise admits no comments at all.

Everything else carries over unchanged. `var` by default and an explicit type when the declared
type carries weight, exactly as in production. camelCase for private fields, parameters, and
locals, with no underscore prefix. One type per file, file scoped namespaces, no XML doc comments,
no em dashes.

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
    var order = orderFaker.Generate();
    var existingOrder = orderFaker.Generate();
    existingOrder.Reference = order.Reference;

    orderRepository.Setup(x => x.Get(It.IsAny<object>())).ReturnsAsync(Result.Ok(existingOrder));

    // Act
    var result = await service.Create(order);

    // Assert
    result.IsFailed.Should().BeTrue();
    result.Errors.Should().Contain(e => e.Message == "An order with this reference already exists");

    orderRepository.Verify(x => x.Get(It.IsAny<object>()), Times.Once);
    orderRepository.Verify(x => x.Insert(It.IsAny<Order>()), Times.Never);
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

When one test file covers several closely related types, as domain and value object tests often
do, prefix the method with the type so the failure names it:

```
Invoice_AddLine_ShouldCalculateTotals_WhenLineIsValid
Money_Addition_ShouldThrow_WhenCurrenciesDiffer
DocumentKey_Generate_ShouldProduceParseableKey
```

Domain tests build their fixtures through small private factory methods with defaulted parameters,
`CreateTestInvoice()` and `CreateTestLine(lineNumber: 2, unitPrice: 500m)`, rather than through a
faker. The values there are chosen so the arithmetic is checkable by hand: a line of exactly 1000
and a rate of exactly 13 percent make the expected 130 obvious in the assertion.

## Parameterized tests

Use `[Theory]` when the variants differ only in input values, and share the same arrangement and
the same assertions. That is the case it was built for, and duplicating it into separate methods
buys nothing.

```csharp
[Theory]
[InlineData("", "Type", "Detail")]
[InlineData("Category", "", "Detail")]
[InlineData("Category", "Type", "")]
public async Task GetByDescriptions_ShouldHandleEmptyParameters_Gracefully(
    string category, string type, string detail)
{
    // Arrange
    var expectedOrders = orderFaker.Generate(1);

    iDbConnection.SetupDapperAsync(c =>
        c.QueryAsync<Order>(
            "GetOrdersByDescriptions",
            It.IsAny<object>(),
            null,
            null,
            CommandType.StoredProcedure
        )).ReturnsAsync(expectedOrders);

    // Act
    var result = await repository.GetByDescriptions(category, type, detail);

    // Assert
    result.Should().NotBeNull();

    context.Verify(x => x.CreateConnection(), Times.Once);
    retryPolicy.Verify(x => x.GetDbPolicy(), Times.Once);
}
```

A second shape of the same case carries the expected result in the row, which reads well when the
method is a classifier and the rows document the classification:

```csharp
[Theory]
[InlineData("Sp", true)]
[InlineData("Unid", false)]
[InlineData("kg", false)]
public void DocumentLine_IsService_ShouldIdentifyCorrectly(string unitOfMeasure, bool expectedIsService)
{
    // Arrange & Act
    var line = CreateTestLine(unitOfMeasure: unitOfMeasure);

    // Assert
    line.IsService.Should().Be(expectedIsService);
}
```

A third case is scale. Same path through the code, different volume, when the behavior under test
is about handling many items rather than about one specific value:

```csharp
[Theory]
[InlineData(1)]
[InlineData(5)]
[InlineData(10)]
public async Task Save_ShouldPersistAllSuccessfully_WhenSavingMultipleOrders(int count)
{
    // Arrange
    var orders = orderFaker.Generate(count);

    // Act
    var tasks = orders.Select(order => repository.Save(order));
    var results = await Task.WhenAll(tasks);

    // Assert
    results.Should().HaveCount(count);
    results.Should().AllSatisfy(key => key.Should().NotBeNullOrEmpty());

    foreach (var order in orders)
    {
        var exists = await repository.Exists(order.Key.Value);
        exists.Should().BeTrue($"Order with key {order.Key.Value} should exist");
    }
}
```

Two things that example does well beyond the parameterization. It asserts on the collection with
`AllSatisfy` instead of looping to assert, and it passes a because message so the failure names
the offending key instead of reporting that false is not true.

The other case `[Theory]` is built for is a guard clause. Every input that the method must reject
belongs in one theory, asserting both the returned value and that the collaborator was never
reached:

```csharp
[Theory]
[InlineData(null)]
[InlineData("")]
public async Task Handle_ShouldReturnNull_WhenReferenceIsMissing(string? orderReference)
{
    // Arrange
    var request = new GetOrderByReferenceQuery(orderReference, Guid.NewGuid());

    // Act
    var result = await handler.Handle(request, CancellationToken.None);

    // Assert
    result.Should().BeNull();

    orderRepository.Verify(r => r.GetByReference(It.IsAny<string>()), Times.Never);
}
```

The `Times.Never` is the point. Without it the test passes for a method that queried the database
with an empty reference and happened to get nothing back.

Use one `[Fact]` per variant when the variants differ in setup, in mock configuration, or in what
gets asserted:

```csharp
Create_ShouldReturnFailure_WhenCustomerDoesNotExist
Create_ShouldReturnFailure_WhenReferenceAlreadyExists
Create_ShouldReturnFailure_WhenTotalIsNegative
```

Those three arrange different mocks and assert different error messages. Folding them into one
`[Theory]` means passing flags or nullable parameters that the test body then branches on to
decide what to assert, which is harder to read than the duplication it avoided.

The rule of thumb is about what the branching decides. Branching to choose which member to invoke,
while every case asserts the same property, is fine:

```csharp
[Theory]
[InlineData("GetAllOrders")]
[InlineData("GetOrderById")]
[InlineData("GetOrdersByCustomer")]
public async Task Repository_ShouldUseCorrectStoredProcedureName_ForEachMethod(string expectedStoredProcName)
{
    // Arrange
    var orders = orderFaker.Generate(1);

    iDbConnection.SetupDapperAsync(c =>
        c.QueryAsync<Order>(expectedStoredProcName, It.IsAny<object>(), null, null, CommandType.StoredProcedure))
        .ReturnsAsync(orders);

    iDbConnection.SetupDapperAsync(c =>
        c.QuerySingleOrDefaultAsync<Order>(expectedStoredProcName, It.IsAny<object>(), null, null, CommandType.StoredProcedure))
        .ReturnsAsync(orders.First());

    // Act & Assert
    switch (expectedStoredProcName)
    {
        case "GetAllOrders":
            await repository.GetAll();
            break;
        case "GetOrderById":
            await repository.GetById(Guid.NewGuid().ToString());
            break;
        case "GetOrdersByCustomer":
            await repository.GetByCustomer("customer-1");
            break;
    }

    context.Verify(x => x.CreateConnection(), Times.Once);
    retryPolicy.Verify(x => x.GetDbPolicy(), Times.Once);
}
```

That test asserts one invariant, that each method reaches the procedure it claims to, across every
method at once. Branching to decide what gets asserted is the case that should have been separate
`[Fact]` methods.

One accepted exception to that: a boolean row that selects between the success path and the
failure path of the same operation, where the pair is the point:

```csharp
[Theory]
[InlineData("USD", "USD", true)]
[InlineData("USD", "CRC", false)]
public void Money_Addition_ShouldRespectCurrency(string currency1, string currency2, bool shouldSucceed)
```

Keep those to two outcomes. Once the flag turns into three or four, the test is a dispatcher and
belongs in separate methods.

One check before adding a case to a theory: does this input take a different path through the
code? If the mock is configured with `It.IsAny<string>()` and every case returns the same thing,
the extra rows add runs, not coverage. Either vary the setup so the cases differ, or drop back to
a single `[Fact]`.

## Assertions and failure modeling

AwesomeAssertions exclusively. Never `Assert.Equal` or any other native xUnit assertion, even
though xUnit is the runner.

```csharp
result.IsSuccess.Should().BeTrue();
result.Value.Id.Should().Be(insertedId);
result.Value.Should().OnlyContain(x => x.CustomerId == customerId);
updatedOrder.Value.UpdatedOn.Should().BeCloseTo(DateTime.UtcNow, TimeSpan.FromMinutes(1));
```

How failure is asserted depends on how the layer signals it, and the two layers signal
differently on purpose.

**Application layer: result objects.** Services and handlers return a result rather than throwing
for an expected failure, so their tests assert on result state and never catch anything. A test
that wants to catch an exception from a service is usually pointing at a service that should have
returned a failure result.

**Domain layer: exceptions.** An entity or value object throws when an invariant is violated,
because there is no caller to hand a result to and no sensible half constructed object to return.
Those tests assert the throw, and assert the message when the message is part of the contract:

```csharp
// Act & Assert
var act = () => invoice.SetCreditTerm(30);
act.Should().Throw<InvalidOperationException>()
   .WithMessage("Credit term can only be set for credit sales");
```

The lambda into a local named `act`, then `Should().Throw<T>()`, is the shape. `Assert.Throws` and
`Record.Exception` are not used. When the whole test is a throw, `// Act & Assert` replaces the
two separate comments.

Do not write custom assertion extensions. Stock fluent chains keep the test vocabulary the same
across every project.

## Test data

Bogus, in one of two shapes depending on what the type needs. Never an inline `Faker<T>` built
inside a test method, and never an object mother.

**Shape one, a faker class in `Tests.Common`,** for types with settable properties that several
test projects share. This is the default.

**Shape two, a `Faker<T>` field configured by a private `SetupFakers()` in the test class,** for
types that only construct through their constructor: aggregates, value objects, anything
immutable. `RuleFor` cannot reach a constructor only type, so these need `CustomInstantiator`,
and the wiring is usually specific enough to one test class that promoting it to `Tests.Common`
buys nothing.

`Tests.Common/Interfaces/IFaker.cs`

```csharp
namespace Acme.Product.Tests.Common.Interfaces;

public interface IFaker<T> where T : class
{
    T Generate();
    List<T> Generate(int count = 5);
}
```

`Tests.Common/DataFakers/FakerBase.cs`

```csharp
using Bogus;
using Acme.Product.Tests.Common.Interfaces;

namespace Acme.Product.Tests.Common.DataFakers;

public abstract class FakerBase<T> : IFaker<T> where T : class
{
    internal Faker<T> faker = null!;

    public T Generate() => faker.Generate();
    public List<T> Generate(int count = 5) => faker.Generate(count);
    public virtual List<T> Generate(int count, int startIndex = 0) => faker.Generate(count);
}
```

`Tests.Common/DataFakers/Entities/OrderFaker.cs`

```csharp
using Bogus;
using Acme.Product.Domain.Entities;

namespace Acme.Product.Tests.Common.DataFakers.Entities;

public class OrderFaker : FakerBase<Order>
{
    public OrderFaker()
    {
        faker = new Faker<Order>()
            .RuleFor(x => x.Id, f => f.Random.Long(1, 999999))
            .RuleFor(x => x.EntityKey, f => Guid.NewGuid())
            .RuleFor(x => x.Reference, f => f.Commerce.Ean13())
            .RuleFor(x => x.Total, f => f.Finance.Amount(10, 5000))
            .RuleFor(x => x.CustomerId, f => f.Random.Long(1, 9999))
            .RuleFor(x => x.CreatedOn, f => f.Date.Past());
    }
}
```

For a constructor only type, the faker lives in the test class and is wired in `SetupFakers()`,
called from the constructor:

```csharp
private void SetupFakers()
{
    issuerFaker = new Faker<Issuer>()
        .CustomInstantiator(f => new Issuer(
            name: f.Company.CompanyName(),
            identification: new Identification(f.PickRandom<IdentificationType>(), f.Random.Replace("##########")),
            commercialName: f.Company.CompanyName(),
            address: GenerateAddress(f),
            contactInfo: GenerateContactInfo(f)));

    invoiceFaker = new Faker<Invoice>()
        .CustomInstantiator(f =>
        {
            var issuer = issuerFaker.Generate();
            var invoice = new Invoice(
                key: DocumentKey.Generate(f.Date.Recent(), issuer.Identification),
                consecutiveNumber: f.Random.Replace("####################"),
                issueDate: f.Date.Recent(),
                issuer: issuer);

            if (f.Random.Bool(0.8f))
            {
                invoice.AddLine(documentLineFaker.Generate());
            }

            return invoice;
        });
}
```

Two details worth copying. Composed fakers call each other rather than duplicating rules, so
`invoiceFaker` generates its issuer through `issuerFaker`. And probabilistic branches inside
`CustomInstantiator`, like adding a tax to eighty percent of lines, keep the generated fixtures
from all being the same happy shape.

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
orderRepository.Setup(x => x.Get(It.IsAny<object>())).ReturnsAsync(Result.Ok(existingOrder));
orderRepository.Verify(x => x.Insert(It.IsAny<Order>()), Times.Once);
```

Callback style setups where the mock has to invoke a delegate it was handed:

```csharp
cacheService
    .Setup(x => x.GetOrCreate(It.IsAny<string>(), It.IsAny<TimeSpan>(), It.IsAny<Func<Task<List<string>>>>()))
    .Returns<string, TimeSpan, Func<Task<List<string>>>>((key, ttl, factory) => factory());
```

**Repositories that own their SQL: mock nothing.** Do not mock `IDbConnection` and do not use a
Dapper mocking package. Hand written SQL is precisely the part a mock cannot verify: a mocked
connection will happily accept a query with a typo in a column name.

**Repositories that only call stored procedures: mock the connection.** When the SQL lives in the
database and the repository is a thin caller, there is nothing to verify by executing it, and the
database may not be reproducible locally at all. Mock `IDbConnection` with a Dapper mocking
package and assert the contract the repository is actually responsible for: the procedure name it
calls, the parameters it passes, the connection lifecycle, and the retry policy. See the
invariant tests below.

**Dependency free services: instantiate the real thing.** A hashing service or a formatter with
no injected dependencies gets tested directly, with the real algorithm running. Same for cheap
in memory infrastructure: use a real `MemoryCache` rather than mocking `IMemoryCache`, and mock
only the logger next to it.

No hand written stubs or fakes implementing an interface. If it is an interface with behavior,
it is a Moq mock. If it is real enough to instantiate cheaply, instantiate it.

## Repository invariant tests

Beyond one test per method, a repository gets a small set of tests that assert properties of the
whole class rather than of a single method. Write these once per repository:

- **Each method reaches the procedure it claims to.** One `[Theory]` over every procedure name,
  dispatching to the matching method, asserting the same lifecycle for all of them. This is the
  branching case allowed above.
- **The connection is disposed after the operation.** Mock the connection as `IDisposable` and
  verify `Dispose` was called exactly once.
- **Every call runs inside the retry policy.** Verify the policy provider was asked for a policy.
- **Repeated calls open the expected number of connections.** Invoke every method in sequence and
  verify the connection factory was called `Times.Exactly(n)`. This catches a method that forgot
  to open its own connection, or one that opens two.

```csharp
[Fact]
public async Task Repository_ShouldEnsureConnectionIsDisposed_AfterOperation()
{
    // Arrange
    var order = orderFaker.Generate();
    var disposableConnection = new Mock<IDbConnection>();
    disposableConnection.As<IDisposable>();

    context.Setup(c => c.CreateConnection()).Returns(disposableConnection.Object);

    disposableConnection.SetupDapperAsync(c =>
        c.ExecuteAsync(
            "InsertOrder",
            It.IsAny<object>(),
            null,
            null,
            CommandType.StoredProcedure
        )).ReturnsAsync(1);

    // Act
    await repository.Insert(order);

    // Assert
    disposableConnection.As<IDisposable>().Verify(x => x.Dispose(), Times.Once);
}
```

These tests are cheap, they are the same shape in every repository, and they fail loudly when
someone adds a method that skips the shared plumbing.

## Repository tests against a real database

A repository that owns its SQL is tested against the real engine, in a container, one container
per test class. Not SQLite standing in for SQL Server, not an in memory provider: the point of the
test is that this exact SQL runs on that exact engine.

```csharp
public class DocumentRepositoryTests : IAsyncLifetime
{
    private readonly SqlServerContainer sqlServerContainer;
    private IDbConnection connection = null!;
    private DocumentRepository repository = null!;

    public DocumentRepositoryTests()
    {
        sqlServerContainer = new SqlServerBuilder()
            .WithPassword("YourStrong!Passw0rd")
            .WithCleanUp(true)
            .Build();

        SetupFakers();
    }

    public async Task InitializeAsync()
    {
        await sqlServerContainer.StartAsync();

        var connectionString = sqlServerContainer.GetConnectionString();
        connection = new SqlConnection(connectionString);
        await connection.OpenAsync();

        await CreateTestSchema();

        var logger = new Mock<ILogger<DocumentRepository>>().Object;
        repository = new DocumentRepository(connection, logger);
    }

    public async Task DisposeAsync()
    {
        connection?.Dispose();
        await sqlServerContainer.DisposeAsync();
    }
}
```

Rules that follow:

- `IAsyncLifetime` for anything the constructor cannot do. The container start, the connection
  open, and the schema creation are all async, and a constructor cannot await. Build the container
  in the constructor, start it in `InitializeAsync`, tear it down in `DisposeAsync`.
- The schema comes from the same migration scripts production uses. A hand maintained copy of the
  schema inside the test project drifts, and a drifted schema turns a real integration test back
  into a mock.
- Isolation comes from the container being per test class, so tests inside one class share state
  and must not depend on ordering. When a test needs a clean slate, it generates its own keys
  rather than truncating tables.
- Assert by reading back. Save, then fetch, then compare the fields that matter. A repository test
  that only checks the return value of `Save` has not proven anything landed.
- Mock the logger and nothing else. Every other collaborator in a repository test is real.

## Controller tests

Instantiate the controller directly with mocked services. No `WebApplicationFactory`, no in
memory host. Set `ControllerContext` by hand when the action reads from the request.

```csharp
[Fact]
public async Task Create_ShouldReturnStatusCode201_WhenSuccessful()
{
    // Arrange
    var request = orderRequestFaker.Generate();
    var createdOrder = orderInfoFaker.Generate();

    orderService.Setup(x => x.Create(It.IsAny<Order>())).ReturnsAsync(Result.Ok(createdOrder));

    // Act
    var result = await controller.Create(request);

    // Assert
    result.Result.Should().BeOfType<CreatedAtActionResult>();

    var createdResult = result.Result as CreatedAtActionResult;
    createdResult!.StatusCode.Should().Be(201);
    createdResult.ActionName.Should().Be(nameof(OrderController.GetById));
    createdResult.Value.Should().Be(createdOrder);

    orderService.Verify(x => x.Create(It.IsAny<Order>()), Times.Once);
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
    logger = new Mock<ILogger<OrderService>>();
    orderRepository = new Mock<IOrderRepository>();
    customerRepository = new Mock<ICustomerRepository>();

    service = new OrderService(logger.Object, orderRepository.Object, customerRepository.Object);
}
```

- No `IClassFixture`, no collection fixtures. Shared state between tests is the thing this
  convention is buying isolation against.
- No DI container in tests. Dependencies are constructed by hand and passed to the constructor.
  If wiring the class under test by hand is painful, the constructor has too many dependencies,
  and that is a design signal worth acting on rather than hiding behind a container.
- Only repository test classes implement `IDisposable`. Mock based tests have nothing to clean up.

## What gets a test

Functional code: anything that makes a decision, transforms state, enforces an invariant, or
talks to a database. Four groups always get a test class:

- **Domain entities and aggregates** that calculate or enforce rules. Totals, tax, discounts,
  state transitions, and every invariant that throws. This is the cheapest and highest value
  testing in the whole solution, because it needs no mocks and no container.
- **Value objects** with operators, factories, or format rules. Currency arithmetic that refuses
  to mix currencies, a key generator with a fixed length and parseable segments.
- **Services and handlers**, with every collaborator mocked.
- **Repositories**, against the real engine or with the connection mocked, depending on who owns
  the SQL.

Controllers get tests when they carry anything beyond delegation: status code selection, route
values, or request shaping.

Declarative code does not get its own test class by default: mappers, DTOs, and validators whose
rules are simple attribute style declarations. They run through the tests above, and testing them
directly mostly asserts that a mapping library still maps. That line moves depending on the
project. A validator holding real branching logic is functional code and belongs in the first
group.

## Coverage

Coverage is collected with coverlet and read as a diagnostic. The practical target is high line
coverage on the functional code above. Whether that number becomes a gate in your pipeline is
your call, not this skill's.

## What not to do

- No MSTest, no NUnit, no second runner alongside xUnit.
- No native xUnit assertions, `Assert.Throws` and `Record.Exception` included. Everything goes
  through the fluent API, throws among them: `act.Should().Throw<T>()`.
- No catching an expected application failure. Assert on the result object instead.
- No `MockBehavior.Strict`.
- No mocked connection in a repository that owns its SQL. Mocking is for repositories that only
  call stored procedures.
- No `Faker<T>` built inline inside a test method. A faker is a class in the shared project or a
  field wired in `SetupFakers()`.
- No assertions on randomly generated values.
- No fixed seeds hiding a test that depends on specific data.
- No hand written stubs where a mock or the real object will do.
- No DI container and no `WebApplicationFactory`.
- No `.Result` and no `.Wait()`. When setup has to be async, that is what `IAsyncLifetime` is for.
- No test class covering unrelated production classes. Closely related domain types can share one
  test class, with the type name prefixing each method.