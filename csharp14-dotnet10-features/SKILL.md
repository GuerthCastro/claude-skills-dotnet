---
name: csharp14-dotnet10-features
description: >
  Reference for what is new in C# 14 and .NET 10, covering both language features and
  platform or library changes. Use this skill whenever writing, reviewing, or migrating C#
  code that targets net10.0, whenever the user asks what is new in C# 14 or .NET 10, and
  when deciding whether a piece of code can use a C# 14 only construct such as the field
  keyword, extension blocks, null conditional assignment, or file based apps. This is a
  feature reference, not a migration runbook.
license: MIT
---

# C# 14 and .NET 10: What Is New

Author: Guerth Castro (github.com/GuerthCastro). Licensed under MIT.

Quick reference for the language and platform features introduced in C# 14, which ships with
the .NET 10 SDK (released November 2025). Use it to confirm a construct is actually available
before suggesting it, and to explain why a modern idiom works the way it does.

This is an original summary of publicly documented features. Verify current behavior against
Microsoft Learn before relying on any detail here, since this area moves quickly and the docs
are the authority.

## When to use this

- Reviewing or writing C# for `<TargetFramework>net10.0</TargetFramework>` and deciding whether
  a C# 14 construct is idiomatic in that spot.
- The user asks what changed between an older C# version and C# 14, or between .NET 9 and 10.
- Deciding whether an old workaround, such as a hand written backing field or a static class
  extension method, can now be simplified.
- Flagging a breaking change during a migration, for example `field` becoming a contextual keyword.

## C# 14 language features

### `field` keyword for auto backed properties

Custom get and set logic without declaring a private backing field. The compiler synthesizes it
and you refer to it through the contextual keyword `field`:

```csharp
public string? FavoritePrimaryColor
{
    get => field;
    set => field = Validate(value);
}
```

Breaking change: an existing accessor that declared a local variable literally named `field`
either fails to compile or changes meaning under the C# 14 compiler, because `field` is now
contextual inside accessors. Fix by renaming the local or writing `@field`.

### Extension blocks

Alongside classic extension methods (a static class plus a `this` parameter), C# 14 adds
`extension(...)` blocks. These declare instance methods, instance properties, static methods,
and static properties or operators against a type, including things that were previously
impossible such as an extension property:

```csharp
public static class IEnumerableOfInt32Extensions
{
    extension(IEnumerable<int> source)
    {
        public IEnumerable<int> WhereGreaterThan(int threshold) =>
            source.Where(x => x > threshold);

        public bool IsEmpty => !source.Any();
    }
}
```

Old style extension methods and new style extension blocks coexist in the same static class,
so adopting the new syntax is optional rather than a breaking change.

### Null conditional assignment

`?.` may now appear on the left hand side of an assignment. The assignment happens only when
the receiver is non null:

```csharp
customer?.Age = newAge; // replaces: if (customer is not null) { customer.Age = newAge; }
```

### Partial members expanded

Through C# 13, `partial` applied to methods, properties, indexers, and events. C# 14 adds
partial events and partial instance constructors. Still not allowed: static constructors,
finalizers, overloaded operators, delegates, and enums.

### `nameof` with unbound generic types

`nameof(List<>)` is now valid and returns `"List"`, instead of requiring a closed type such as
`nameof(List<string>)`.

### File based apps

A single `.cs` file can be executed with no project file and no directory structure, which
suits scripts, prototypes, and teaching:

```bash
dotnet run app.cs
```

Top level statements are supported. This targets quick scripting, not a replacement for
project based applications.

## .NET 10 platform and library features

- Noun first `dotnet` CLI aliases. Microsoft now recommends the noun first form of CLI commands
  for clearer scripting and documentation. The older verb first forms still work.
- Numeric string ordering. A comparison mode that orders embedded numbers numerically rather
  than lexicographically, so `item2` sorts before `item10`.
- LINQ `LeftJoin` and `RightJoin`. New operators alongside `Join` and `GroupJoin`, replacing the
  old `GroupJoin` plus `DefaultIfEmpty` plus `SelectMany` workaround.
- Asynchronous ZIP archive APIs for reading and writing.
- JSON Patch (RFC 6902) for ASP.NET Core, implemented on `System.Text.Json`.
- A first party OpenAPI document generator for ASP.NET Core Minimal APIs, usable in place of or
  alongside Swashbuckle style tooling.
- Central Package Management, managing NuGet versions across a solution through
  `Directory.Packages.props` instead of pinning per project.

## Scope notes

- This file covers C# 14 and .NET 10 specifically. It is not a cumulative history of the
  language.
- Feature availability can depend on SDK version and language version settings, not only on the
  target framework. Check `<LangVersion>` before assuming a construct compiles.
- For migration mechanics such as project files, CI pipelines, and package feeds, use a
  migration runbook. This skill answers "does this feature exist and how does it work".