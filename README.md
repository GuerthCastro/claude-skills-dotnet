# .NET 10 Skills

Opinionated Claude skills for .NET 10 development, written and maintained by
[Guerth Castro](https://github.com/GuerthCastro): C# conventions, SOLID and Clean Architecture
review, and a C# 14 / .NET 10 feature reference.

Author: Guerth Castro. License: MIT.

## Skills

| Skill | What it does |
| --- | --- |
| [`dotnet10-conventions`](./dotnet10-conventions) | Strict C# and .NET 10 conventions: one type per file, file scoped namespaces, primary constructors, no `var`, Dapper only, attribute driven entities, soft deletes, LookUp catalog pattern, Clean Architecture layout. |
| [`solid-review`](./solid-review) | Audits C#, TypeScript, or Angular code against the five SOLID principles plus Clean Architecture layering, and returns a severity ranked report with refactorings. |
| [`csharp14-dotnet10-features`](./csharp14-dotnet10-features) | Feature reference for C# 14 and .NET 10: `field` keyword, extension blocks, null conditional assignment, partial constructors, file based apps, LINQ `LeftJoin`, Central Package Management. |

## Installation

Quick version, for Claude Code on every project:

```bash
git clone https://github.com/GuerthCastro/claude-skills-dotnet.git
mkdir -p ~/.claude/skills
cp -r claude-skills-dotnet/{dotnet10-conventions,solid-review,csharp14-dotnet10-features} ~/.claude/skills/
```

For per project installs, the Claude apps, verification, and troubleshooting, see
[INSTALL.md](./INSTALL.md).

Each skill is a folder containing a single `SKILL.md` with YAML frontmatter. Claude loads the
`description` to decide when the skill applies, so keep it intact if you fork.

## A note on opinions

`dotnet10-conventions` is deliberately strict. Rules like banning `var` and Entity Framework are
personal choices that have paid off in the codebases I maintain, not universal truths. Fork it
and change the rules you disagree with. `solid-review` and `csharp14-dotnet10-features` are
close to neutral and should be useful as they are.
