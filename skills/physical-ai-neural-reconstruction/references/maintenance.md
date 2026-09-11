# Keeping This Router Up to Date

The upstream `nurec-index` skill (at
<https://github.com/NVIDIA/nurec-skills/blob/main/skills/nurec-index/SKILL.md>)
is hand-curated by the NRS team. When it adds or restructures
sibling skills:

1. Add a row to `Pick a skill` in `SKILL.md` for any new use case.
2. Add a row to `Sibling skills (upstream)` in `SKILL.md`.
3. If the new skill changes a multi-step pipeline, update
   `references/workflows.md` — keep the workflow letters aligned with
   the upstream `nurec-index` `references/workflows.md` (currently
   A–G) so cross-references stay usable.
4. Re-verify the upstream URLs, container names, and release pins
   still match each sibling's frontmatter `metadata:` block:
   - `ncore-data-conversion` — <https://github.com/NVIDIA/ncore>. The
     **skill itself** now ships from that repo too
     (`skills/ncore-data-conversion/`). NCore publishes semver tags, not
     a `2026.04`-style release
   - `nre` — `nvcr.io/nvidia/nre/nre-ga` +
     `nvcr.io/nvidia/nre/nre-tools-ga`, NRE `release_26.04`
   - `asset-harvester` — <https://github.com/NVIDIA/asset-harvester>,
     `nvidia/asset-harvester` on Hugging Face. The **skill itself** now
     ships from that repo too (`skills/asset-harvester/`).
   - `harmonizer` — <https://github.com/NVIDIA/harmonizer>,
     `nvidia/Harmonizer` (public), base image
     `nvcr.io/nvidia/pytorch:25.10-py3`. The **skill itself** now ships
     from that repo too (`skills/harmonizer/`).
   - `physical-ai-datasets` — `nvidia/PhysicalAI-*` on Hugging Face
5. If the upstream renames a sibling skill (e.g. `nurec-fixer` →
   `harmonizer`), a container channel (`nre` → `nre-ga`), or a model
   repo (`nvidia/DiffusionHarmonizer` → `nvidia/Harmonizer`), search
   this skill for the old name and update every **active route or
   path** — the picker table, workflow steps, sibling skills table,
   mix-ups, hard rules, and troubleshooting. Deliberately keep the old
   name where it is history rather than a route: `former_name`, the
   discovery aliases, the stale-copy warning, and the stale-names
   troubleshooting row.
6. Check whether the upstream layout still roots skills at
   `skills/<name>/SKILL.md` (with `.agents/skills` as a symlink); if
   it moves, update `metadata.upstream` and
   `references/upstream-fetch.md`.

Treat the upstream `nurec-index` as authoritative **for the routing
taxonomy and workflow ordering**; this skill mirrors only the picker
tables, the workflow ordering, and the upstream fetch recipe. It is not
authoritative for `asset-harvester`, `harmonizer` or
`ncore-data-conversion`, which are maintained in their own product repos.
