# Plugin-Based Skill Distribution for Standalone Apps

| Field | Value |
|-------|-------|
| **Status** | Proposal |
| **Author** | @matgren |
| **Created** | 2026-04-16 |
| **Related** | [2026-04-02-empty-app-starter-presets.md](./2026-04-02-empty-app-starter-presets.md), [create-app AGENTS.md](../../packages/create-app/AGENTS.md) |

## TLDR

`create-mercato-app` currently stamps 11 frozen skill copies into every scaffolded app. These skills never update — even when the developer runs `yarn upgrade @open-mercato/*`, the `.ai/skills/` files remain at the version they were when the app was created. This spec proposes replacing frozen copies with a plugin-based distribution model where skills are installed via the Claude Code plugin marketplace and stay in sync through `/plugin marketplace update`.

## Problem Statement

### Skills drift from day one

When a developer scaffolds an app with `create-mercato-app`, 11 AI skills are copied into `.ai/skills/`:

```
backend-ui-design, code-review, data-model-design, eject-and-customize,
implement-spec, integration-builder, integration-tests, module-scaffold,
spec-writing, system-extension, troubleshooter
```

These skills are source files committed to the app's repo. When OM ships improvements to these skills (bug fixes, new patterns, updated conventions), the app never receives them. The npm packages update via `yarn upgrade`, but `.ai/skills/` is untouched.

### The problem compounds over time

- OM core develop currently has **26 skills** (vs 11 in create-app agentic), including `auto-create-pr`, `auto-review-pr`, `auto-qa-scenarios`, `smart-test`, and others that standalone app developers never receive.
- Skills reference `AGENTS.md` conventions that evolve. A skill written against March conventions may give wrong guidance in June.
- The `code-review` skill references a `review-checklist.md` with rules that change as OM's architecture evolves. Frozen checklists miss new rules.
- Multiple apps scaffolded at different times run different skill versions, creating inconsistent developer experience across the OM ecosystem.

### Claude Code doesn't discover skills from `node_modules/`

Claude Code loads skills from: personal `~/.claude/skills/`, project `.claude/skills/`, installed plugins, and `--add-dir` directories. It does **not** scan `node_modules/`. This means there is no way to ship skills via npm packages and have them auto-discovered.

The only mechanism that keeps skills fresh without manual copying is the **plugin marketplace**.

## Proposed Solution

### 1. Recommend plugin installation during agentic setup

Update the `create-mercato-app` agentic wizard to recommend installing the [om-superpowers](https://github.com/SHGrowth/om-superpowers) Claude Code plugin:

```
🤖  Agentic workflow setup

   Which AI coding tool will you use with this project?

   1. Claude Code     (Anthropic)
   2. Codex           (OpenAI)
   3. Cursor          (Anysphere)
   4. Multiple tools  (select individually)
   5. Skip — set up manually later

   Enter number(s) separated by comma [1]: 1

   📦  OM Skills Plugin

   Open Mercato skills are distributed via the om-superpowers plugin.
   This keeps skills in sync as the platform evolves.

   Install now? [Y/n]: y

   Run this in your project:
     claude /plugin marketplace add SHGrowth/om-superpowers

   Skills will auto-update via: /plugin marketplace update
```

### 2. Skip `.ai/skills/` copy when plugin is chosen

When the developer confirms plugin installation:
- Do NOT copy skills from `agentic/shared/ai/skills/` into the app
- The plugin provides all skills (synced daily from OM core develop)
- Still copy `.ai/qa/`, `.ai/specs/`, and other non-skill agentic content

### 3. Keep frozen copies as fallback

When the developer skips plugin installation or uses Codex/Cursor (where plugins aren't available):
- Continue copying skills as today (frozen fallback)
- Add a comment to the generated `AGENTS.md`: "Skills were copied at scaffold time. For auto-updating skills, install the om-superpowers plugin."

### 4. Retire skills from `agentic/shared/ai/skills/` over time

As the plugin ecosystem matures and becomes the standard approach:
- Phase out the frozen copies from `create-app/agentic/`
- Reduce maintenance burden of keeping two distribution channels
- Timeline: after plugin adoption reaches meaningful levels (measured by installs/usage)

## What the plugin provides beyond frozen copies

The [om-superpowers](https://github.com/SHGrowth/om-superpowers) plugin is not just a mirror of OM core skills. It adds:

| Feature | Frozen copies | Plugin |
|---------|--------------|--------|
| **Skill count** | 11 | 20 (including auto-create-pr, auto-review-pr, auto-continue-pr) |
| **Updates** | Never (frozen at scaffold time) | Daily sync from OM core develop, delivered via `/plugin marketplace update` |
| **AGENTS.md references** | None (developer reads `node_modules/` compiled JS) | Vendored AGENTS.md files from all OM packages, refreshed on sync |
| **Orchestration** | None — skills are standalone | Piotr (CTO persona) orchestrates spec writing → implementation → code review pipeline |
| **Product management** | None | Marty Cagan persona for business requirements and App Spec creation |
| **UX review** | None | Steve Krug persona for UI architecture review |
| **User proxy** | None | Answers routine agent questions on behalf of the developer |
| **Platform capability discovery** | Static — developer reads compiled JS | Live — `gh search code` + `.ai/specs/implemented/` + Task Router |

## Architecture

### Sync pipeline

```
OM core develop (.ai/skills/)
        ↓ daily GitHub Action
om-superpowers plugin (skills/)
        ↓ /plugin marketplace update
Developer's Claude Code session
```

The plugin's [sync workflow](https://github.com/SHGrowth/om-superpowers/blob/main/.github/workflows/sync-om-skills.yml) runs daily, pulls skills from two OM sources:
- `.ai/skills/` — core skills (implement-spec, code-review, spec-writing, etc.)
- `packages/create-app/agentic/shared/ai/skills/` — app-building skills (module-scaffold, system-extension, etc.)

If changes are detected, it opens a PR with a patch version bump. After merge, users receive updates on next `/plugin marketplace update`.

### Preset compatibility

The plugin works with all `create-mercato-app` presets (`classic`, `empty`, `crm`). Skills are platform-level tooling — they work regardless of which modules are enabled.

## Changes to `create-mercato-app`

### Modified files

| File | Change |
|------|--------|
| `packages/create-app/src/setup/tools/claude-code.ts` | Add plugin recommendation prompt after Claude Code is selected |
| `packages/create-app/src/setup/wizard.ts` | Pass plugin choice to tool generators |
| `packages/create-app/agentic/shared/AGENTS.md.template` | Add note about plugin-based skill updates |

### No breaking changes

- `create-mercato-app` without flags continues to work exactly as today (classic preset, frozen skills)
- The plugin recommendation is opt-in during the interactive wizard
- `--skip-agentic-setup` still works — no skills copied, no plugin recommended
- Imported ready apps (`--app`, `--app-url`) are unaffected — they skip the agentic wizard entirely

## Migration for existing apps

Existing apps with frozen `.ai/skills/` can upgrade:

```bash
# Install the plugin
claude /plugin marketplace add SHGrowth/om-superpowers

# Optionally remove the frozen copies (plugin takes precedence)
rm -rf .ai/skills/
```

The plugin's skills are namespaced with `om-` prefix and take precedence over project-level skills with the same name.

## Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Plugin unavailable (GitHub down, registry issue) | Low | Frozen copies remain as fallback; not deleted |
| Plugin skills diverge from OM conventions | Low | Daily automated sync from OM core develop; same source of truth |
| Developer confusion: "which skills am I using?" | Medium | `AGENTS.md` clearly states source; `/skills` command shows loaded skills |
| Codex/Cursor users can't use plugins | Medium | Frozen copies continue for non-Claude-Code tools; no regression |

## Implementation Plan

1. Add plugin recommendation prompt to `packages/create-app/src/setup/tools/claude-code.ts`
2. Skip `.ai/skills/` copy when plugin is chosen
3. Update `AGENTS.md.template` with plugin installation note
4. Update `packages/create-app/AGENTS.md` to document the new flow
5. Test: scaffold with plugin → verify no `.ai/skills/` copied, plugin instruction shown
6. Test: scaffold without plugin → verify frozen copies still work (no regression)

## Changelog

- 2026-04-16: Initial proposal — plugin-based skill distribution for standalone apps
