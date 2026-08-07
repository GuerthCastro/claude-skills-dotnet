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

```csharp
// Program.cs
using Acme.Site.Abstractions;
using Acme.Site.Extensions;
using Acme.Site.Options;
using Acme.Site.Services;

WebApplicationBuilder Builder = WebApplication.CreateBuilder(args);

Builder.Configuration.AddKeyVaultSecretsWhenConfigured();

Builder.Services.AddControllersWithViews();

Builder.Services.AddValidatedOptions<ApiOptions>(Builder.Configuration.GetSection(ApiOptions.SectionName));
Builder.Services.AddValidatedOptions<EmailOptions>(Builder.Configuration.GetSection(EmailOptions.SectionName));

Builder.Services.AddTransient<IOrderRepository, OrderRepository>();

WebApplication App = Builder.Build();

if (!App.Environment.IsDevelopment())
{
    App.UseExceptionHandler("/Home/Error");
    App.UseHsts();
}

App.UseHttpsRedirection();
App.UseRouting();
App.UseAuthorization();
App.MapStaticAssets();

App.MapControllerRoute(name: "default", pattern: "{controller=Home}/{action=Index}/{id?}").WithStaticAssets();

App.Run();
```

Registration order matters in exactly one place: any configuration source added to
`Builder.Configuration` wins over the sources added before it. `CreateBuilder` has already added
`appsettings.json`, the environment override, user secrets in Development, and environment
variables. Anything appended after that outranks all of them.

## Configuration

The naive port reads keys inline, which is what most migration examples show:

```csharp
string BaseUrl = Configuration["APIs:Billing"] ?? throw new InvalidOperationException("Missing configuration: APIs:Billing");
```

This works and it is still wrong for a codebase anyone else will touch. The key is a string
repeated across files with no compiler help, the failure surfaces on the first request that needs
the value rather than at deployment, and a typo in the JSON is indistinguishable from a missing
deployment setting.

Bind a class per section instead. The section name lives as a constant on the type it describes:

```csharp
// Options/EmailOptions.cs
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

```csharp
// Extensions/ServiceCollectionExtensions.cs
namespace Acme.Site.Extensions;

public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddValidatedOptions<TOptions>(this IServiceCollection Services, IConfiguration Section) where TOptions : class
    {
        Services.AddOptions<TOptions>().Bind(Section).ValidateDataAnnotations().ValidateOnStart();

        return Services;
    }

    public static IServiceCollection AddOptionsValidatedOnFirstUse<TOptions>(this IServiceCollection Services, IConfiguration Section) where TOptions : class
    {
        Services.AddOptions<TOptions>().Bind(Section).ValidateDataAnnotations();

        return Services;
    }
}
```

Consumers take `IOptions<T>` and the magic strings disappear:

```csharp
public class HomeController(IOptions<EmailOptions> EmailOptions) : Controller
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

```csharp
// Services/EmailService.cs
public class EmailService(HttpClient Client, IOptions<TestingOptions> TestingOptions) : IEmailService
{
    public async Task<bool> SendAsync(string To, string Body, CancellationToken CancellationToken)
    {
        TestingOptions Testing = TestingOptions.Value;
        string Recipient = Testing.InTesting ? Testing.TestEmail : To;

        using HttpResponseMessage Response = await Client.PostAsync($"send?to={Recipient}", null, CancellationToken);

        return Response.IsSuccessStatusCode;
    }
}
```

Conditional rules go in `IValidatableObject`, which `ValidateDataAnnotations` honours:

```csharp
// Options/TestingOptions.cs
using System.ComponentModel.DataAnnotations;

namespace Acme.Site.Options;

public sealed class TestingOptions : IValidatableObject
{
    public bool InTesting { get; set; }

    public string TestEmail { get; set; } = string.Empty;

    public IEnumerable<ValidationResult> Validate(ValidationContext ValidationContext)
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

```csharp
// Extensions/ConfigurationManagerExtensions.cs
using Azure.Identity;
using Acme.Site.Options;

namespace Acme.Site.Extensions;

public static class ConfigurationManagerExtensions
{
    public static ConfigurationManager AddKeyVaultSecretsWhenConfigured(this ConfigurationManager Configuration)
    {
        KeyVaultOptions? KeyVault = Configuration.GetSection(KeyVaultOptions.SectionName).Get<KeyVaultOptions>();

        if (string.IsNullOrWhiteSpace(KeyVault?.Uri))
        {
            return Configuration;
        }

        DefaultAzureCredentialOptions CredentialOptions = new();

        if (!string.IsNullOrWhiteSpace(KeyVault.TenantId))
        {
            CredentialOptions.TenantId = KeyVault.TenantId;
        }

        Configuration.AddAzureKeyVault(new Uri(KeyVault.Uri), new DefaultAzureCredential(CredentialOptions));

        return Configuration;
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

```csharp
// Controllers/HomeController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Options;
using Acme.Site.Abstractions;
using Acme.Site.Models.ViewModels;
using Acme.Site.Options;

namespace Acme.Site.Controllers;

public class HomeController(IOrderRepository OrderRepository, IOptions<EmailOptions> EmailOptions) : Controller
{
    [HttpGet]
    public async Task<IActionResult> Index(CancellationToken CancellationToken)
    {
        IReadOnlyList<Order> Orders = await OrderRepository.GetRecentAsync(CancellationToken);

        IndexViewModel Model = new()
        {
            Orders = Orders.ToList()
        };

        return View("Index", Model);
    }

    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Submit(IndexViewModel Model, CancellationToken CancellationToken)
    {
        if (!ModelState.IsValid)
        {
            return View("Index", Model);
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

```csharp
// Models/ViewModels/IndexViewModel.cs
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
    @foreach (RoleModel Role in Model.Roles)
    {
        <option value="@Role.RoleId">@Role.Role</option>
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
Builder.Services.AddTransient<IOrderRepository, OrderRepository>();
```

Split by subject rather than by table. A repository per screen ends up duplicating queries; a
repository per concept does not.

## API access

An external API gets a Service, registered as a typed `HttpClient`. Resolve options through the
provider so the base address comes from validated configuration:

```csharp
Builder.Services.AddHttpClient<IBillingService, BillingService>((Provider, Client) =>
{
    ApiOptions Apis = Provider.GetRequiredService<IOptions<ApiOptions>>().Value;

    Client.BaseAddress = new Uri(Apis.Billing);
});
```

A Service returns models, exactly like a repository. Returning `HttpResponseMessage` pushes
transport concerns into the controller, which then inspects `IsSuccessStatusCode` and owns a
lifetime it did not create:

```csharp
// Services/BillingService.cs
public class BillingService(HttpClient Client, ILogger<BillingService> Logger) : IBillingService
{
    public async Task<bool> SubmitAsync(string Reference, CancellationToken CancellationToken)
    {
        using HttpResponseMessage Response = await Client.PostAsync($"submit?reference={Reference}", null, CancellationToken);

        if (!Response.IsSuccessStatusCode)
        {
            Logger.LogWarning("Billing submit returned {Status}.", (int)Response.StatusCode);
        }

        return Response.IsSuccessStatusCode;
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
5. Replace every configuration read with a bound options class.
6. Move secrets to user secrets, then to Key Vault, then rotate them.

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
- No XML doc comments.
- No em dashes in comments or documentation.
- No unrequested edits bundled into a requested change.
