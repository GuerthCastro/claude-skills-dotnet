# Installation

Each skill is a folder with one `SKILL.md` inside. Nothing to compile, no dependencies. You are
copying folders into a directory the agent already watches.

Pick your tool:

- [Claude Code](#claude-code) (terminal, IDE extensions, desktop Code tab)
- [Claude apps](#claude-apps-web-desktop-mobile) (web, desktop, mobile)
- [Other agents](#other-agents)

---

## Claude Code

Two locations, chosen by scope:

| Location | Scope | Use when |
| --- | --- | --- |
| `~/.claude/skills/` | Every project on your machine | These are your personal conventions. |
| `.claude/skills/` | One repository | The team should get them from source control. |

Worth knowing before you pick: when the same skill name exists in both, **personal wins over
project**. That is the opposite of what most people assume, and it is the usual explanation for
editing a skill in a repo and seeing no change in behavior. Enterprise policy, in turn,
overrides personal.

Skills also load from nested `.claude/skills/` directories below your working directory, so a
package inside a monorepo can ship skills that only apply while Claude is working on that
package.

### Step 1: get the files

```bash
git clone https://github.com/GuerthCastro/claude-skills-dotnet.git
cd claude-skills-dotnet
```

**Verify:** `ls` shows `dotnet10-conventions`, `dotnet10-testing`, `solid-review`, and
`csharp14-dotnet10-features`, each containing a `SKILL.md`.

**No git on the machine:** use the green Code button on GitHub, download the ZIP, extract it. The
folder structure is what matters, not how it got there.

### Step 2: copy the skills into place

Personal, available in every project:

```bash
mkdir -p ~/.claude/skills
cp -r dotnet10-conventions dotnet10-testing solid-review csharp14-dotnet10-features ~/.claude/skills/
```

Per project, committed with the repository:

```bash
mkdir -p /path/to/your/project/.claude/skills
cp -r dotnet10-conventions dotnet10-testing solid-review csharp14-dotnet10-features /path/to/your/project/.claude/skills/
```

Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills"
Copy-Item -Recurse dotnet10-conventions,dotnet10-testing,solid-review,csharp14-dotnet10-features "$HOME\.claude\skills\"
```

**Verify:** the tree looks exactly like this, with `SKILL.md` one level below the skill folder:

```
~/.claude/skills/
├── csharp14-dotnet10-features/
│   └── SKILL.md
├── dotnet10-conventions/
│   └── SKILL.md
├── dotnet10-testing/
│   └── SKILL.md
└── solid-review/
    └── SKILL.md
```

When a skill does not load, this is almost always why. The two ways to get it wrong:

- One level too deep: `~/.claude/skills/claude-skills-dotnet/solid-review/SKILL.md`. Move the
  skill folders up.
- The `SKILL.md` copied without its folder: `~/.claude/skills/SKILL.md`. A bare `SKILL.md` with
  no parent folder is not scanned. The folder name is the skill identity.

### Step 3: confirm the agent sees them

**Verify:** open Claude Code in a .NET project and ask it to review a class for SOLID compliance,
or ask what is new in C# 14. The right skill should be picked up on its own, because that is what
its `description` is written for.

**If nothing happens:**

1. Restart Claude Code. Cheap, and it rules out anything related to when the directory appeared.
2. Check the frontmatter: the file opens with `---`, contains `name` and `description`, and closes
   with `---`. A tab character or an unquoted colon inside the description invalidates the block
   and the skill is skipped silently.
3. Confirm the `name` in the frontmatter matches the folder name.
4. Ask for the skill by name. If it works when named but never triggers on its own, installation
   is fine and the issue is the wording of the description.
5. If you installed per project and nothing changed, check whether an older copy of the same skill
   is sitting in `~/.claude/skills/`. Personal wins.

One caveat that catches people: Cowork sessions and cloud sessions do not read `~/.claude/skills/`
from your machine. They load the skills enabled for your account, and cloud sessions additionally
load project skills committed to the cloned repository. A personal only install will look missing
there.

---

## Claude apps (web, desktop, mobile)

Custom skills are uploaded as ZIP files through **Settings > Features**, on the Pro, Max, Team,
and Enterprise plans with code execution enabled. Check that first, since without it the section
below does not apply.

### Step 1: build the archives

One ZIP per skill, with the skill folder at the root of the archive:

```bash
zip -r dotnet10-conventions.zip dotnet10-conventions
zip -r dotnet10-testing.zip dotnet10-testing
zip -r solid-review.zip solid-review
zip -r csharp14-dotnet10-features.zip csharp14-dotnet10-features
```

On Windows, right click the folder and choose Send to > Compressed (zipped) folder.

**Verify:** `unzip -l solid-review.zip` lists `solid-review/SKILL.md`. If it lists a bare
`SKILL.md` with no folder prefix, you zipped the contents instead of the folder. Redo it from the
parent directory.

### Step 2: upload

Settings > Features > custom skills, then upload each ZIP.

**Verify:** each skill appears in the list with the description shown in this repository.

**If an upload is rejected:** the apps validate more strictly than Claude Code does. In practice
the cause is almost always the archive structure from step 1, so check that before rewriting any
frontmatter.

---

## Other agents

`SKILL.md` is an open format, and other agents read it directly. Consult that tool for its skills
directory, since the path differs. Nothing in these skills depends on Claude specific
features: no scripts, no bundled resources, no tool restrictions, just Markdown.

---

## Updating

```bash
cd claude-skills-dotnet
git pull
cp -r dotnet10-conventions dotnet10-testing solid-review csharp14-dotnet10-features ~/.claude/skills/
```

Better, if you plan to follow the repo: symlink instead of copying, and `git pull` becomes the
whole update procedure.

```bash
ln -s "$(pwd)/solid-review" ~/.claude/skills/solid-review
```

On Windows, the equivalent is `mklink /D` from an elevated prompt, or `New-Item -ItemType
SymbolicLink` in PowerShell with Developer Mode enabled.

## Uninstalling

```bash
rm -rf ~/.claude/skills/dotnet10-conventions \
       ~/.claude/skills/dotnet10-testing \
       ~/.claude/skills/solid-review \
       ~/.claude/skills/csharp14-dotnet10-features
```

In the Claude apps, remove them from Settings > Features.

## Reference

The official documentation is the authority whenever something here goes stale:

- Agent Skills overview: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- Skills in Claude Code: https://code.claude.com/docs/en/skills