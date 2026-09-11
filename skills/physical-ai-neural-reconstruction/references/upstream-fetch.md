# Locating and Fetching Upstream Skills

The canonical NuRec router (named `nurec-index`) and two of its five
sibling skills live in `https://github.com/NVIDIA/nurec-skills` under
`skills/<name>/SKILL.md`. **`asset-harvester`, `harmonizer` and
`ncore-data-conversion` are the exceptions** — all three are maintained in
their own product repos; see the next section.

The `nurec-skills` repo also exposes `.agents/skills` as a
symlink onto `skills/`, so both paths resolve to the same tree. Refer
to a sibling skill by its `name:` (e.g. `nre`) — that name is portable
across agent runtimes that implement the `agentskills.io` standard.
The folder name always matches the skill `name:` (e.g. the `nre` skill
lives at `skills/nre/`).

## Siblings fetched from their own repos

These three ship from their product repos and are maintained there:

| skill | repo | path |
|---|---|---|
| `asset-harvester` | [`NVIDIA/asset-harvester`](https://github.com/NVIDIA/asset-harvester) | `skills/asset-harvester/` |
| `harmonizer` (was `nurec-fixer`) | [`NVIDIA/harmonizer`](https://github.com/NVIDIA/harmonizer) | `skills/harmonizer/` |
| `ncore-data-conversion` (was `ncore`) | [`NVIDIA/ncore`](https://github.com/NVIDIA/ncore) | `skills/ncore-data-conversion/` |

**Do not read any of these from a `nurec-skills` checkout.** Those copies still
exist but are no longer updated, so reading them silently yields stale
guidance instead of failing — the `nurec-fixer` copy, for instance, still
says `nvidia/Harmonizer` needs a Hugging Face token, which it does not.

Fetch these as you would any other upstream. **[Ask before
cloning](#ask-before-cloning)** applies — each is a network fetch plus a
local write. Companion files sit beside `SKILL.md` **where present**:
`ncore-data-conversion` ships `references/` but no `scripts/`.

## Where to look on the local disk (try in order)

This covers the two `nurec-skills`-hosted siblings **and** `harmonizer`
and `ncore-data-conversion`: both were renamed, so a local hit under either
new name cannot be a stale copy — those are still called `nurec-fixer` and
`ncore`. **Never fall back to `nurec-fixer` or `ncore`.**

`asset-harvester` is excluded — its name did not change, so a local copy of
it may well be the stale `nurec-skills` version.

1. `.agents/skills/<name>/SKILL.md` (Cursor, Codex, NemoClaw)
2. `.claude/skills/<name>/SKILL.md` (Claude Code)
3. `.cursor/skills/<name>/SKILL.md` (project-scoped)
4. `~/.cursor/skills/<name>/SKILL.md` (personal skills)
5. An existing `nurec-skills` clone under the shared upstream root.

## Ask before cloning

> **Do not auto-clone.** The step below performs a **network fetch**
> of an external GitHub repository **and writes to the local
> filesystem**. That can violate organizational network/security
> policy and exposes the user to supply-chain risk if the upstream is
> ever tampered with. Show the user the exact command and get explicit
> consent (e.g. "OK to `git clone
> https://github.com/NVIDIA/nurec-skills` into `<DIR>`?") **before**
> running it. Prefer a pinned tag or SHA over `HEAD`, prefer fetching only
> the needed `SKILL.md` when the layout allows it, and never silently
> default to `/tmp`. **Exception — `ncore-data-conversion`:** no current
> NCore release tag contains that skill, so use `main` or a commit at or
> after `b5d3b8a`; pinning the latest tag returns a 404 for its path.
>
> If the user declines, stop and report which sibling skill is
> missing. Do not fall back to silent network access.

## Clone or refresh the upstream (`nurec-skills` siblings only)

This clones `nurec-skills` and therefore serves only the two siblings
hosted there. `harmonizer`, `ncore-data-conversion` and `asset-harvester`
come from their own product repos — see the section above — and must never be taken from a
`nurec-skills` clone.

Use the shared upstream root unless the user has set a NuRec-specific
override:

```bash
UPSTREAM_ROOT="${NUREC_SKILLS_UPSTREAM_ROOT:-${PHYSICAL_AI_SKILL_HUB_UPSTREAM_ROOT:-$HOME/.physical-ai-skill-hub/upstreams}}"
mkdir -p "$UPSTREAM_ROOT"
if [ -d "$UPSTREAM_ROOT/nurec-skills/.git" ]; then
  git -C "$UPSTREAM_ROOT/nurec-skills" fetch --tags
  git -C "$UPSTREAM_ROOT/nurec-skills" checkout main
  git -C "$UPSTREAM_ROOT/nurec-skills" pull --ff-only
else
  # Only after the user has agreed to the clone.
  git clone --depth 1 https://github.com/NVIDIA/nurec-skills.git \
    "$UPSTREAM_ROOT/nurec-skills"
fi
test -f "$UPSTREAM_ROOT/nurec-skills/skills/nurec-index/SKILL.md"
```

Then read the upstream skill before running any mutating command:

```bash
# Upstream router (table of contents), name: nurec-index
cat "$UPSTREAM_ROOT/nurec-skills/skills/nurec-index/SKILL.md"

# Sibling skills (replace <folder> per the table in SKILL.md):
cat "$UPSTREAM_ROOT/nurec-skills/skills/<folder>/SKILL.md"
```

Skills that pin a specific upstream commit ship the actual file under
`skills/<folder>/_versions/<branch>/<commit>/SKILL.md` with a
top-level `<folder>/SKILL.md` symlink to the currently-selected
version. Follow the symlink; don't hand-pick a `_versions/` path
unless the user asked for a specific revision.

Companion files (`references/`, `scripts/`, `assets/`) live next to
**the sibling skill's** `SKILL.md`, not next to this router. Open the
sibling skill first and follow its References section.
