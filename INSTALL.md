# Installation

These skills follow the Agent Skills open standard: each one is a folder containing a single
`SKILL.md`. There is nothing to compile and no dependencies to install. You are copying folders
into a directory the agent already watches.

Pick the section for your tool:

- [Claude Code](#claude-code) (terminal, IDE extensions, desktop Code tab)
- [Claude apps](#claude-apps-web-desktop-mobile) (web, desktop, mobile)
- [Other agents](#other-agents)

---

## Claude Code

Claude Code discovers skills automatically from two locations. Choose based on scope:

| Location | Scope | Use when |
| --- | --- | --- |
| `~/.claude/skills/` | Every project on your machine | These are your personal conventions. |
| `.claude/skills/` | One repository | The team should get them from source control. |

Project skills load from `.claude/skills/` in the directory where you start Claude Code and in
every parent directory up to the repository root, so starting Claude in a subfolder still picks
them up.

### Step 1: get the files

```bash
git clone https://github.com/GuerthCastro/claude-skills-dotnet.git
cd claude-skills-dotnet
```

If you only want the skills branch:

```bash
git clone -b skills/dotnet10 --single-branch https://github.com/GuerthCastro/claude-skills-dotnet.git
```

**Verify:** `ls` shows `dotnet10-conventions`, `solid-review`, and `csharp14-dotnet10-features`,
each containing a `SKILL.md`.

**If it fails:** on a machine without git, use the green Code button on GitHub, download the ZIP,
and extract it. The folder structure is what matters, not how it got there.

### Step 2: copy the skills into place

Personal, available in every project:

```bash
mkdir -p ~/.claude/skills
cp -r dotnet10-conventions solid-review csharp14-dotnet10-features ~/.claude/skills/
```

Per project, committed with the repository:

```bash
mkdir -p /path/to/your/project/.claude/skills
cp -r dotnet10-conventions solid-review csharp14-dotnet10-features /path/to/your/project/.claude/skills/
```

On Windows PowerShell:

```powershell
New-Item -ItemType Directory -Force "$HOME\.claude\skills"
Copy-Item -Recurse dotnet10-conventions,solid-review,csharp14-dotnet10-features "$HOME\.claude\skills\"
```

**Verify:** the tree looks exactly like this, with `SKILL.md` one level below the skill folder:

```
~/.claude/skills/
├── csharp14-dotnet10-features/
│   └── SKILL.md
├── dotnet10-conventions/
│   └── SKILL.md
└── solid-review/
    └── SKILL.md
```

**If the skill does not load later, this is almost always the reason.** Two common mistakes:

- An extra nesting level: `~/.claude/skills/claude-skills-dotnet/solid-review/SKILL.md`. Move the
  skill folders up one level.
- The `SKILL.md` copied without its folder: `~/.claude/skills/SKILL.md`. The folder name is part
  of the skill identity, so it has to stay.

### Step 3: confirm the agent sees them

Claude Code watches the skill directories for changes, so adding a skill under `~/.claude/skills/`
or a project `.claude/skills/` is picked up inside the current session without a restart.

One exception: if the top level `skills` directory did not exist when your session started,
restart Claude Code so it begins watching the new directory. If you just ran `mkdir -p` in
step 2, that is your situation. Restart.

**Verify:** open Claude Code in a .NET project and ask it to review a class for SOLID compliance,
or ask what is new in C# 14. The relevant skill should be picked up on its own, because that is
what its `description` is written for.

**If nothing happens:**

1. Restart Claude Code. Covers the new-directory case above.
2. Check the YAML frontmatter of the skill: the file must open with `---`, contain `name` and
   `description`, and close with `---`. A tab character or a stray unquoted colon inside the
   description will invalidate the block.
3. Confirm the `name` in the frontmatter matches the folder name.
4. Ask for the skill by name explicitly. If it works when named but not on its own, the skill is
   installed correctly and the issue is trigger wording, not installation.

---

## Claude apps (web, desktop, mobile)

Custom skills are uploaded as ZIP files through **Settings > Features**. One ZIP per skill, with
the skill folder at the root of the archive.

### Step 1: build the archives

```bash
zip -r dotnet10-conventions.zip dotnet10-conventions
zip -r solid-review.zip solid-review
zip -r csharp14-dotnet10-features.zip csharp14-dotnet10-features
```

On Windows, right click the folder and choose Send to > Compressed (zipped) folder.

**Verify:** `unzip -l solid-review.zip` lists `solid-review/SKILL.md`. If it lists a bare
`SKILL.md` with no folder prefix, you zipped the contents instead of the folder. Redo it from the
parent directory.

### Step 2: upload

Settings > Features > custom skills, then upload each ZIP.

**Verify:** the skill appears in the list with the description shown in this repository.

**If the upload is rejected:** the Skills API applies stricter frontmatter limits than Claude Code.
`name` is capped at 64 lowercase and hyphenated characters and cannot contain the reserved words
`anthropic` or `claude`; `description` is capped at 1024 characters. All three skills here are
inside those limits, so a rejection usually means the ZIP has the wrong internal structure.

---

## Other agents

The `SKILL.md` format is an open standard, so other agents that read it can use these files
directly. Consult that tool's documentation for its skills directory. Nothing in these three
skills depends on Claude specific features: no scripts, no bundled resources, no tool
restrictions, just Markdown.

---

## Updating

```bash
cd claude-skills-dotnet
git pull
cp -r dotnet10-conventions solid-review csharp14-dotnet10-features ~/.claude/skills/
```

Claude Code picks up edits to `SKILL.md` inside the current session, so no restart is needed for
an update to a skill that was already installed.

An alternative worth knowing: a skill entry in the personal or project location can be a symlink
to a directory elsewhere on disk, and Claude Code follows it. That lets you point at your clone
instead of copying, so `git pull` is the whole update procedure:

```bash
ln -s "$(pwd)/solid-review" ~/.claude/skills/solid-review
```

## Uninstalling

```bash
rm -rf ~/.claude/skills/dotnet10-conventions \
       ~/.claude/skills/solid-review \
       ~/.claude/skills/csharp14-dotnet10-features
```

In the Claude apps, remove them from Settings > Features.

## Reference

Official documentation, which is the authority if any detail here goes stale:

- Agent Skills overview: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- Skills in Claude Code: https://code.claude.com/docs/en/skills
