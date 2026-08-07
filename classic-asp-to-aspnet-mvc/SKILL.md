---
name: classic-asp-to-aspnet-mvc
description: >
  Conventions and a working playbook for porting Classic ASP or Web Forms sites to ASP.NET Core
  MVC on net10.0: Program.cs composition, controllers and routing, view models, Razor layouts,
  sections and partial views, wwwroot assets, configuration through validated options instead of
  raw configuration keys, secrets in user secrets locally and Azure Key Vault in production,
  Dapper repositories behind Abstractions interfaces, and typed HttpClient services that return
  models rather than transport objects. Use this skill whenever migrating a Classic ASP, ASP, or
  Web Forms application to ASP.NET Core MVC, reviewing such a migration, scaffolding the MVC
  layout for a ported site, or deciding where ported inline logic belongs in the new structure.
license: MIT
---

# Classic ASP to ASP.NET Core MVC

The target is a server side rendered MVC app on `net10.0`. Most of the page is computed on the
server and returned as finished HTML, which is the closest analogue to how the ASP pages already
work. That similarity is what makes the port tractable, and it is also the trap: an `.asp` file
translated line by line into a `.cshtml` file compiles and runs while keeping every structural
problem it had.

Three components:

- **Models** act like types. Fields and their datatypes, passed around as a unit.
- **Views** are `.cshtml` files. Mostly HTML, with C# available where it earns its place.
- **Controllers** hold the logic. Methods take parameters and return status codes, JSON, or Views.

## Non-negotiable rules

- Every value read from configuration goes through a bound options class. No
  `Configuration["Some:Key"]` anywhere, including bootstrap code that runs before the container
  exists: that reads a section, and the section name is a constant on the options type it describes.
- Every string that is an external contract is a named constant: third party form fields, HTTP
  header names, query string keys. The literal appears once, next to the type that owns it.
- One statement per line. A call that fits on one line stays on one line, however long. Object and
  collection initializers and constructor declarations are exempt and keep their braces open.
- Do not port anything whose purpose you cannot state in one sentence. If it survives the port
  because nobody knew what it did, it will outlive the next three developers for the same reason.
- Every options class is validated at startup with `ValidateOnStart`, except options whose values
  are only meaningful at a specific operation.
- No secret in `appsettings.json`. User secrets locally, Key Vault in production.
- Services and repositories return models. Never leak `HttpResponseMessage`, `IDataReader`, or
  `DataTable` past their own boundary.
- Every controller action carries an explicit HTTP verb attribute.
- Every form that mutates state carries `[ValidateAntiForgeryToken]`.
- Application JavaScript goes in `@section Scripts`. Only assets that must resolve before body
  render belong in `@section Head`.
- Preserve the existing URLs. A migrated site inherits inbound links, bookmarks, and search
  rankings that a new route template silently breaks.

## Program.cs

The composition root, in two halves. First the builder, where services are registered. Then the
app, where middleware and routes are declared. Nothing else belongs here.

`Program.cs`

```csharp
using Acme.Site.Abstractions;
using Acme.Site.Extensions;
using Acme.Site.Options;
using Acme.Site.Services;

var builder = WebApplication.CreateBuilder(args);

builder.Configuration.AddKeyVaultSecretsWhenConfigured();

builder.Services.AddControllersWithViews();

builder.Services.AddValidatedOptions<ApiOptions>(builder.Configuration.GetSection(ApiOptions.SectionName));
builder.Services.AddValidatedOptions<EmailOptions>(builder.Configuration.GetSection(EmailOptions.SectionName));

builder.Services.AddTransient<IOrderRepository, OrderRepository>();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseRouting();
app.UseAuthorization();
app.MapStaticAssets();

app.MapControllerRoute(name: "default", pattern: "{controller=Home}/{action=Index}/{id?}").WithStaticAssets();

app.Run();
```

Registration order matters in exactly one place: any configuration source added to
`builder.Configuration` wins over the sources added before it. `CreateBuilder` has already added
`appsettings.json`, the environment override, user secrets in Development, and environment
variables. Anything appended after that outranks all of them.

## Configuration

The naive port reads keys inline, which is what most migration examples show:

```csharp
string baseUrl = Configuration["APIs:Billing"] ?? throw new InvalidOperationException("Missing configuration: APIs:Billing");
```

This works and it is still wrong for a codebase anyone else will touch. The key is a string
repeated across files with no compiler help, the failure surfaces on the first request that needs
the value rather than at deployment, and a typo in the JSON is indistinguishable from a missing
deployment setting.

Bind a class per section instead. The section name lives as a constant on the type it describes:

`Options/EmailOptions.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace Acme.Site.Options;

public sealed class EmailOptions
{
    public const string SectionName = "Email";

    [Required]
    [EmailAddress]
    public string FromEmail { get; set; } = string.Empty;

    [Required]
    public string Subject { get; set; } = string.Empty;
}
```

Register it once, validated, through an extension that makes the two behaviours distinguishable
by name:

`Extensions/ServiceCollectionExtensions.cs`

```csharp
namespace Acme.Site.Extensions;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddValidatedOptions<TOptions>(this IServiceCollection services, IConfiguration section) where TOptions : class
    {
        services.AddOptions<TOptions>().Bind(section).ValidateDataAnnotations().ValidateOnStart();

        return services;
    }

    public static IServiceCollection AddOptionsValidatedOnFirstUse<TOptions>(this IServiceCollection services, IConfiguration section) where TOptions : class
    {
        services.AddOptions<TOptions>().Bind(section).ValidateDataAnnotations();

        return services;
    }
}
```

Consumers take `IOptions<T>` and the magic strings disappear:

```csharp
public class HomeController(IOptions<EmailOptions> emailOptions) : Controller
```

Data annotations buy more than presence checks. `[EmailAddress]`, `[Url]`, and `[Range]` catch
values that are present but malformed, which is the failure mode that survives a naive port:
`GetValue<double>` returns `0` for garbage input, so a misconfigured score threshold approves
everything instead of failing.

### When not to validate on start

`ValidateOnStart` is correct for values the app cannot serve a single request without. It is wrong
for values that only matter inside one operation, because startup validation runs whether or not
that operation is ever reached.

Watch the resolution path, not the intent. A service injected into a controller constructor is
built on every request to that controller, so reading `IOptions<T>.Value` in its constructor
validates on every page load. If the option only matters at send time, hold the `IOptions<T>` and
read `.Value` inside the method:

`Services/EmailService.cs`

```csharp
public class EmailService(HttpClient client, IOptions<TestingOptions> testingOptions) : IEmailService
{
    public async Task<bool> Send(string to, string body, CancellationToken cancellationToken)
    {
        var testing = testingOptions.Value;
        string recipient = testing.InTesting ? testing.TestEmail : to;

        using var response = await client.PostAsync($"send?to={recipient}", null, cancellationToken);

        return response.IsSuccessStatusCode;
    }
}
```

Conditional rules go in `IValidatableObject`, which `ValidateDataAnnotations` honours:

`Options/TestingOptions.cs`

```csharp
using System.ComponentModel.DataAnnotations;

namespace Acme.Site.Options;

public sealed class TestingOptions : IValidatableObject
{
    public bool InTesting { get; set; }

    public string TestEmail { get; set; } = string.Empty;

    public IEnumerable<ValidationResult> Validate(ValidationContext validationContext)
    {
        if (InTesting && string.IsNullOrWhiteSpace(TestEmail))
        {
            yield return new ValidationResult($"{nameof(TestEmail)} is required while {nameof(InTesting)} is true.", new[] { nameof(TestEmail) });
        }
    }
}
```

## Secrets

`appsettings.json` is committed and safe for non-sensitive settings: base URLs, subject lines,
feature flags, a reCAPTCHA site key that the browser receives anyway. Nothing else.

Locally, secrets go in user secrets, keyed off `UserSecretsId` in the csproj. In Visual Studio,
right click the project and choose Manage User Secrets. In VS Code, the command palette has
`.NET: Manage User Secrets`.

In production, Azure Key Vault. Add `Azure.Extensions.AspNetCore.Configuration.Secrets` and
`Azure.Identity`, then register the provider conditionally so a developer without vault access can
still run the app:

`Extensions/ConfigurationManagerExtensions.cs`

```csharp
using Azure.Identity;
using Acme.Site.Options;

namespace Acme.Site.Extensions;

public static class ConfigurationManagerExtensions
{
    public static ConfigurationManager AddKeyVaultSecretsWhenConfigured(this ConfigurationManager configuration)
    {
        var keyVault = configuration.GetSection(KeyVaultOptions.SectionName).Get<KeyVaultOptions>();

        if (string.IsNullOrWhiteSpace(keyVault?.Uri))
        {
            return configuration;
        }

        DefaultAzureCredentialOptions credentialOptions = new();

        if (!string.IsNullOrWhiteSpace(keyVault.TenantId))
        {
            credentialOptions.TenantId = keyVault.TenantId;
        }

        configuration.AddAzureKeyVault(new Uri(keyVault.Uri), new DefaultAzureCredential(credentialOptions));

        return configuration;
    }
}
```

The vault URI is not a secret. Put it in `appsettings.Production.json` so the environment selects
it, or in an App Service setting if the artifact must stay environment agnostic.

Key Vault secret names cannot contain a colon. The default secret manager maps `--` to `:`, so
`Recaptcha:Secret` is stored as `Recaptcha--Secret`.

Four things to know before the first deployment:

- The App Service needs Managed Identity enabled and the **Key Vault Secrets User** role on the
  vault. Without the role, startup fails with a 403 that looks like an outage, not a config error.
- `AddAzureKeyVault` has no optional mode. An unreachable vault means the app does not start. For
  required secrets that is correct, but it should be a decision, not a surprise.
- `IOptions<T>` is a singleton, so rotating a secret in the vault needs a restart. Live rotation
  requires `IOptionsMonitor<T>` and a `ReloadInterval`.
- Any developer with a local credential that can read the vault can run the app in Production mode
  and pull production secrets to their machine. The real control is the role assignment.

Moving a secret out of `appsettings.json` does not remove it from git history. Rotate anything that
was ever committed, then load the new value into the vault.

## Controllers

One controller per section of the site. Give it a route prefix with `[Route("segment")]` at the
class, or leave it to the conventional route from `Program.cs` when the ported URLs already match.

Every action gets an explicit verb attribute. It documents the route and it stops a GET action from
answering a POST:

`Controllers/HomeController.cs`

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Options;
using Acme.Site.Abstractions;
using Acme.Site.Models.ViewModels;
using Acme.Site.Options;

namespace Acme.Site.Controllers;

public class HomeController(IOrderRepository orderRepository, IOptions<EmailOptions> emailOptions) : Controller
{
    [HttpGet]
    public async Task<IActionResult> Index(CancellationToken cancellationToken)
    {
        var orders = await orderRepository.GetRecent(cancellationToken);

        IndexViewModel model = new()
        {
            Orders = orders.ToList()
        };

        return View("Index", model);
    }

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Submit(IndexViewModel model, CancellationToken cancellationToken)
    {
        if (!ModelState.IsValid)
        {
            return View("Index", model);
        }

        return RedirectToAction(nameof(Thanks));
    }
}
```

`return View()` without a name resolves to the action name. Naming it explicitly costs one string
and survives a rename of the action, which is the common case during a migration when actions get
renamed away from their `.asp` filenames.

Use `nameof` for redirect targets. A renamed action then breaks the build instead of producing a
404 in production.

### Rebinding a view model on redisplay

Any property the controller populates rather than the form must be repopulated before returning
the view on a validation failure, because model binding only fills what the request carried. Mark
those properties `[BindNever]` so a crafted POST cannot set them.

## View models

A view model is a normal model kept in `Models/ViewModels/`. It carries everything the view needs
and nothing else:

`Models/ViewModels/IndexViewModel.cs`

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Mvc.ModelBinding;

namespace Acme.Site.Models.ViewModels;

public class IndexViewModel
{
    [Required(ErrorMessage = "Email Address is required.")]
    [EmailAddress(ErrorMessage = "Please enter a valid email address.")]
    [StringLength(254)]
    public string LeadEmail { get; set; } = string.Empty;

    [BindNever]
    public string RecaptchaSiteKey { get; set; } = string.Empty;
}
```

`required` on a property is right for a read only view model built entirely by the controller. It
is wrong for a form bound one, because the controller can no longer construct it without setting
every member.

## Views

`@model` at the top names the view model, `@Model` reaches it anywhere in the file:

```cshtml
@model IndexViewModel
@{
    ViewData["Title"] = "Orders";
}

<span class="currentPage">@Model.Page</span>

<select name="SelectedRoleId" required>
    @foreach (var role in Model.Roles)
    {
        <option value="@role.RoleId">@role.Role</option>
    }
</select>
```

`ViewData["Title"]` sets the browser tab title, read by `_Layout.cshtml`.

### Layout, sections and partials

`_Layout.cshtml` is the shell every view renders inside. Global CSS, global scripts, and anything
shared across pages belong there. It exposes named sections:

```cshtml
@await RenderSectionAsync("Head", required: false)
@RenderBody()
@await RenderSectionAsync("Scripts", required: false)
```

Application JavaScript goes in `@section Scripts`, which renders at the end of the body. Only
assets that must resolve earlier belong in `@section Head`: third party tag managers, analytics
pixels, and scripts a page element references by callback name at render time.

Partial views are the component model. Name them with a leading underscore and load them with
`@await Html.PartialAsync("_Analytics")`.

## wwwroot

Public static assets: CSS, JavaScript, fonts, images. No C# here. Reference them with `~/` and add
`asp-append-version="true"` so a deployed change busts the cache.

## Data access

A repository holds the queries and returns models that mirror the shape of the result. Interfaces
live in `Abstractions/`, one per file, and every repository is registered in `Program.cs`:

```csharp
builder.Services.AddTransient<IOrderRepository, OrderRepository>();
```

Split by subject rather than by table. A repository per screen ends up duplicating queries; a
repository per concept does not.

## API access

An external API gets a Service, registered as a typed `HttpClient`. Resolve options through the
provider so the base address comes from validated configuration:

```csharp
builder.Services.AddHttpClient<IBillingService, BillingService>((provider, client) =>
{
    var apis = provider.GetRequiredService<IOptions<ApiOptions>>().Value;

    client.BaseAddress = new Uri(apis.Billing);
});
```

A Service returns models, exactly like a repository. Returning `HttpResponseMessage` pushes
transport concerns into the controller, which then inspects `IsSuccessStatusCode` and owns a
lifetime it did not create:

`Services/BillingService.cs`

```csharp
public class BillingService(HttpClient client, ILogger<BillingService> logger) : IBillingService
{
    public async Task<bool> Submit(string reference, CancellationToken cancellationToken)
    {
        using var response = await client.PostAsync($"submit?reference={reference}", null, cancellationToken);

        if (!response.IsSuccessStatusCode)
        {
            logger.LogWarning("Billing submit returned {Status}.", (int)response.StatusCode);
        }

        return response.IsSuccessStatusCode;
    }
}
```

Return `bool` for operations with no payload, a model or `IReadOnlyList<T>` when there is one.
Decide deliberately whether a failed call throws or degrades: `EnsureSuccessStatusCode` is right
when the page cannot render without the data, and wrong when a notification failing should not
cost the user their submission.

## Migration playbook

Order that keeps the site working throughout:

1. Scaffold the MVC project and port `_Layout.cshtml` first. Get one page rendering with the real
   CSS before touching logic.
2. Port pages as Views with hardcoded data. Confirm the markup renders, then wire the model.
3. Move inline ASP database calls into repositories, one screen at a time.
4. Move inline API calls into Services.
5. Replace every configuration read with a bound options class, validated at startup.
6. Move every secret out of `appsettings.json` into user secrets. Local development keeps working
   and nothing sensitive is committed from here on.
7. Wire Key Vault for the deployed environments:
   1. Add `Azure.Extensions.AspNetCore.Configuration.Secrets` and `Azure.Identity`.
   2. Add a `KeyVaultOptions` class carrying `Uri`, `TenantId`, and `ManagedIdentityClientId`.
   3. Register the provider on the first line after `CreateBuilder`, conditional on a vault URI
      being present, so a developer without vault access still runs the app on user secrets.
   4. Put the vault URI in `appsettings.Production.json`, never in `appsettings.json`. The URI and
      the tenant id are not secrets, so they are committed.
   5. Create the secrets in the vault with `--` in place of every `:`.
   6. Enable Managed Identity on the App Service and grant it **Key Vault Secrets User** on the
      vault. Leave `ManagedIdentityClientId` empty unless the identity is user assigned.
8. Rotate every secret that was ever committed, then load the new values into the vault. Emptying
   the file does not remove the value from git history.
9. Verify the three startup paths before deploying.

The verification in step 9 is what catches a half configured vault, and it is worth doing
explicitly rather than assuming:

- Development with no vault URI: the app starts on user secrets and never reaches Azure.
- Target environment with the vault reachable: the app starts, and a value that exists only in the
  vault renders on a page.
- Target environment with one secret removed: the app refuses to start and names the missing
  option. If it starts anyway, the option is not registered with `ValidateOnStart`.

Things that consistently bite:

- **Vendor snippets.** Tag managers and tracking pixels arrive as blocks with their own comment
  delimiters. They belong in a partial, loaded from `_Layout`, and they usually need to stay in
  `<head>`. Moving them to the end of the body changes when they fire.
- **Query string flags.** Ported pages often carry a parameter nothing in the code reads, because
  the consumer is an analytics container configured outside the repository. Confirm before deleting
  it. Name the constant so the next reader is not asking the same question.
- **Silent anti-bot paths.** A failed check that redirects to the success page is usually
  deliberate, so an attacker learns nothing. Preserve the behaviour and make sure the marker that
  distinguishes those hits survives the port.
- **Form field names.** Third party widgets post fixed field names such as `g-recaptcha-response`.
  They are external contracts. Put them in constants, never rename them.
- **URLs.** Conventional routing on `{controller=Home}/{action=Index}/{id?}` reproduces most ported
  paths. Adding route templates to make URLs prettier breaks inbound links.
- **An incomplete vault.** The list of secrets someone reports having created is not evidence. Boot
  the app against the vault with startup validation on, because that names anything missing in one
  line and costs less than reading a portal screen carefully.

## What not to do

- No `Configuration["Key"]` literal, anywhere, for any reason.
- No secret in any committed file.
- No `HttpResponseMessage` or `DataTable` crossing a service or repository boundary.
- No action without an explicit HTTP verb attribute.
- No state changing form without `[ValidateAntiForgeryToken]`.
- No application JavaScript in `@section Head`.
- No route template added purely for aesthetics on a migrated URL.
- No magic string at a call site where a named constant belongs.
- No statement wrapped across lines when it fits on one, initializers and constructors aside.
- No XML doc comments and no inline comments, apart from `// Arrange`, `// Act`, and `// Assert`
  in a test body.
- No underscore prefix on a private field, and no `Async` or `Sync` suffix on a method you wrote.
- No em dashes in comments or documentation.
- No unrequested edits bundled into a requested change.
