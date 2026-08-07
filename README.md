# .NET Agent Skills

You already told the agent how you write code. Yesterday, in another chat, in detail. Then the
session ended and it went back to suggesting Entity Framework and a repository per screen.

These are five skills that make that explanation permanent. Drop them in a directory and the
agent picks them up on its own, in the right moment, without you pasting a wall of rules into
every conversation.

## What a skill is

A folder with one `SKILL.md` inside. Markdown, plus a bit of YAML at the top saying what the
skill covers and when it applies. The agent reads that header, decides whether the current task
matches, and loads the body only if it does.

No install, no dependencies, no build step. If you can read Markdown, you can audit every line
of what your agent is about to follow, which is more than can be said for most tooling that
touches your code.

## The five skills

| Skill | What it does |
| --- | --- |
| [`dotnet10-conventions`](./dotnet10-conventions) | The rules I actually work by: one type per file, file scoped namespaces, primary constructors, camelCase locals and fields with no underscore prefix, plain Dapper with hand written SQL, versioned migration scripts, `long` keys with a `Guid` for the outside world, soft deletes, a single LookUp table for every catalog, Clean Architecture layering that is enforced rather than described. |
| [`classic-asp-to-aspnet-mvc`](./classic-asp-to-aspnet-mvc) | The playbook for porting a Classic ASP or Web Forms site to ASP.NET Core MVC without carrying its structure across: where inline logic goes, options classes instead of raw configuration keys, user secrets locally and Key Vault in production, and the ordering that keeps the site working through the whole migration. |
| [`solid-review`](./solid-review) | Turns "review this" into an actual audit. Five principles, one verdict each, violations ranked Critical over Major over Minor, and a refactored version for anything serious. It also checks layering, because a class can satisfy all five principles and still have a handler reaching into `HttpContext`. |
| [`dotnet10-testing`](./dotnet10-testing) | How the tests get written: xUnit with `[Theory]` reserved for variants that differ only in input values, Moq for collaborators, Bogus fakers as real classes in a shared test project, AwesomeAssertions, result objects instead of thrown exceptions, and repository tests that hit the real engine in a disposable container rather than a mocked connection. |
| [`csharp14-dotnet10-features`](./csharp14-dotnet10-features) | Stops the agent from writing 2019 C# in a `net10.0` project. The `field` keyword, extension blocks, null conditional assignment, partial constructors, file based apps, LINQ `LeftJoin`, Central Package Management, and the breaking change that bites anyone who ever named a local variable `field`. |

## Install

```bash
git clone https://github.com/GuerthCastro/claude-skills-dotnet.git
mkdir -p ~/.claude/skills
cp -r claude-skills-dotnet/{dotnet10-conventions,classic-asp-to-aspnet-mvc,dotnet10-testing,solid-review,csharp14-dotnet10-features} ~/.claude/skills/
```

That covers every project on your machine. For per project installs, the Claude apps, other
agents, and the troubleshooting for when a skill refuses to load, see [INSTALL.md](./INSTALL.md).

One thing worth knowing before you fork: the `description` in the frontmatter is not
documentation, it is the trigger. It is the only part the agent reads before deciding whether
the skill is relevant. Rewrite the body freely. Touch the description carefully.

## About the opinions

`dotnet10-conventions` is not a survey of best practices. It is what I do, and some of it is
genuinely arguable.

Banning Entity Framework means writing SQL by hand, which is slower on day one and predictable on
day four hundred, when a query plan matters more than a fluent API. Banning comments sounds
absolute until you notice how often a comment is a rename that never happened, which is why the
only ones left are `// Arrange`, `// Act`, and `// Assert`, and those mark structure rather than
explain code.

Two rules used to say the opposite of what I do, and they are worth naming because a rule that
contradicts the code it governs is worse than no rule. The skill banned `var` outright and asked
for PascalCase locals; both came from a codebase I no longer maintain. `var` is now the default,
with an explicit type where the declared type carries weight, and locals, parameters, and private
fields are camelCase with no underscore.

If a rule does not fit your team, delete it. Nothing here is load bearing for the other rules,
and a fork with your name on it is worth more than a config file you argue with.

`dotnet10-testing` is opinionated in the same way, but it has a different pedigree: it was
extracted from a real suite of 675 tests rather than written from memory, so every rule in it is
one I already live with.

`solid-review` and `csharp14-dotnet10-features` are close to neutral. Those two should be useful
as they are.

## Contributing

Issues and pull requests are welcome, particularly for the feature reference, since .NET moves
faster than any single person can track. For the conventions skill, expect me to be stubborn
about the rules and receptive about the examples.

## License and author

MIT. Written and maintained by [Guerth Castro](https://github.com/GuerthCastro).
Use them, fork them, sell what you build with them. Just leave the copyright notice in place.