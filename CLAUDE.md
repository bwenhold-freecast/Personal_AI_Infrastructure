# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is the **public, sanitized** Personal AI Infrastructure (PAI) distribution repo. It is *not* a runtime checkout — users do not run PAI from here. Instead, this repo produces and ships two artifact shapes:

- `Releases/v{X.Y.Z}/.claude/` — complete `.claude/` directories meant to be copied into a user's `$HOME` and activated by running `./install.sh` inside the copied tree. v5.0.0 is the current shipping release.
- `Packs/{SkillName}/` — standalone, AI-installable capabilities that drop a single skill (and any deps) into an existing `~/.claude/skills/` without requiring full PAI. Each pack is its own install wizard.

The private upstream (referenced internally as "Kai" / `PAI_DIRECTORY`) is the source of truth; this repo is the sanitized public mirror. **Never copy blindly from the private tree to here** — read `SECURITY.md` before any cross-tree transfer.

## Do not confuse the two CLAUDE.md files

- **This file** (`/CLAUDE.md`) — guides contributors editing this distribution repo.
- **`Releases/v5.0.0/.claude/CLAUDE.md`** — the *runtime* CLAUDE.md that ships to end users; it defines modes (NATIVE / ALGORITHM / MINIMAL), voice endpoints, the Algorithm contract, and is parsed by Claude Code on session start in a real PAI install. Treat it like product output: edit it deliberately, never to encode contributor-side conventions.

If a rule is about *making a release or pack*, it belongs here. If it's about *how PAI behaves when running*, it belongs in the runtime CLAUDE.md inside the relevant release.

## Common commands

All TypeScript runs on **Bun** — never `npm` / `npx` / `node`. There is no package.json at repo root; each tool is a standalone Bun script invoked by path.

```bash
# Validate that no protected/forbidden content is about to be committed.
# Used by the pre-commit hook automatically; run manually before any push.
bun Tools/validate-protected.ts
bun Tools/validate-protected.ts --staged   # staged files only

# Back up / restore a local ~/.claude install (operates on the user's runtime, not this repo).
bun Tools/BackupRestore.ts backup
bun Tools/BackupRestore.ts backup --name "pre-v5"
bun Tools/BackupRestore.ts list
bun Tools/BackupRestore.ts restore <backup-name>

# Manual install of the current release into your own ~/.claude (testing a release end-to-end).
cd Releases/v5.0.0
cp -R .claude ~/
cd ~/.claude && ./install.sh
```

There is no `test`, `lint`, or `build` step at repo root — releases are static trees, packs are static trees, and `Tools/*.ts` are leaf scripts. CI is two GitHub Actions in `.github/workflows/` (`claude.yml`, `claude-code-review.yml`) that run Claude on PRs; they are not a substitute for local validation.

## The protected-files firewall

`/.pai-protected.json` is the contract that defines what must (or must not) appear in this repo. It is enforced by `Tools/validate-protected.ts`, which is wired into the pre-commit hook. It has three classes of rules:

- **Required files** — e.g. `README.md`, `SECURITY.md` must exist and must contain "PAI" or "Personal AI Infrastructure".
- **Forbidden directories** — patterns like `^skills/`, `^MEMORY/`, `^History/`, `^context/`, `^progress/` must never appear at repo root (those are private-tree artifacts).
- **Forbidden patterns** — regex list catching API keys, OAuth tokens, AWS keys, personal emails, internal hostnames, and other secrets across staged content.

**When you add a new sensitive pattern or new private path, add it to `.pai-protected.json` — don't rely on documentation alone.** The validator is the only thing that runs by default.

## Repo layout

```
/
├─ Packs/{SkillName}/              # Standalone installable skill packs (v5 ships 45+)
│   ├─ README.md                   #   What it does and why
│   ├─ INSTALL.md                  #   5-phase wizard the DA executes
│   ├─ VERIFY.md                   #   Post-install checklist
│   └─ src/                        #   Verbatim copy target → ~/.claude/skills/{Name}/
├─ Releases/v{X.Y.Z}/              # Full ready-to-drop .claude trees
│   ├─ .claude/                    #   Copied to user's $HOME
│   │   ├─ install.sh              #     Bash bootstrap → hands off to TS installer
│   │   ├─ settings.json           #     Permissions, env, hooks wiring
│   │   ├─ CLAUDE.md               #     RUNTIME guidance (see warning above)
│   │   ├─ PAI/                    #     ALGORITHM, DOCUMENTATION, MEMORY, PULSE, TOOLS, USER
│   │   ├─ skills/                 #     45 skills as separate dirs + skills/CLAUDE.md guard
│   │   ├─ hooks/                  #     37 lifecycle hooks
│   │   ├─ agents/, commands/      #
│   │   └─ test-results/
│   └─ README.md                   #   Release notes
├─ Tools/
│   ├─ validate-protected.ts       # Pre-commit safety check (see firewall section)
│   └─ BackupRestore.ts            # Manage ~/.claude backups
├─ .pai-protected.json             # The deny-list manifest
├─ PLATFORM.md                     # macOS ✅ / Linux ✅ / Windows ❌ status matrix
└─ SECURITY.md                     # Public-vs-private boundary rules
```

## Editing rules for the major surfaces

**Editing a pack under `Packs/{Name}/src/`**
Treat the `src/` directory as the literal skill layout that lands at `~/.claude/skills/{Name}/`. Anything you add must obey PAI's skill conventions: `SKILL.md` at the root, then flat `Workflows/`, `Tools/`, `References/` (max 2 levels deep). If you're scaffolding, renaming, or restructuring a skill, the runtime guard at `Releases/v5.0.0/.claude/skills/CLAUDE.md` says to invoke the `CreateSkill` skill rather than handrolling — that rule applies inside a runtime install, but the *same canonical shape* must be respected when authoring the pack here.

**Editing a release under `Releases/v{X.Y.Z}/.claude/`**
Anything inside a release tree is a frozen distribution artifact. Edits land verbatim on every fresh install. Bumping versions, modifying `settings.json`, or rewriting docs inside a release directly affects new users. Prefer additive edits to the *latest* release; older releases (v2.3 – v4.0.3) are kept as historical references and should generally not be modified.

**Editing `Tools/*.ts`**
These run against the user's environment, not just CI. `validate-protected.ts` is on the pre-commit hot path — be careful with exit codes and stdout shape. Both scripts are standalone Bun files (no shared lib); cross-tool helpers belong inline rather than as new modules.

## Conventions worth knowing

- **TypeScript + Bun only.** No Python, no Node, no npm. `.ts` files have a `#!/usr/bin/env bun` shebang and are runnable directly.
- **Paths must portable.** Never hardcode `/Users/...` or any personal directory. Use `${HOME}`, `$PAI_DIR`, `import.meta.dir`, or paths relative to the script. `PLATFORM.md` tracks every known platform-specific footgun.
- **Heavy bias toward plain text / Markdown.** Avoid SQLite, JSON-as-database, or any opaque store inside releases or packs — the principle (from `README.md`) is "if you can't `cat` it, we don't want it." `.pai-protected.json` is an intentional exception because it's a config schema, not data.
- **The public/private boundary is structural, not vibes-based.** Forbidden directory patterns in `.pai-protected.json` are listed because they exist privately. If you find yourself wanting to add one of those directory names at the repo root, you're almost certainly importing the wrong thing.

## Release/pack workflow (high level)

1. Land changes against the *current* release (`Releases/v5.0.0/`) and/or relevant packs in `Packs/`.
2. Run `bun Tools/validate-protected.ts --staged` to confirm no protected content slipped in.
3. Commit; the pre-commit hook re-runs the validator. If it fires, **fix the underlying leak** — don't bypass with `--no-verify`.
4. For a new release version, add a new `Releases/v{X.Y.Z}/` directory rather than mutating an existing one; update `Releases/README.md` to describe what changed.

When in doubt about whether something is safe to commit, check `SECURITY.md` first, then `.pai-protected.json`, then ask.
