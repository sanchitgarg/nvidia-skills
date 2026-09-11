---
name: physical-ai-neural-reconstruction
description: "Router for NVIDIA NuRec/NRE: USDZ rendering, NCore conversion, 3DGS, gRPC sensor sim, carline adaptation, PhysicalAI HF datasets. Do NOT use for SimReady or infra setup."
license: Apache-2.0
version: "0.4.0"
tools:
  - Read
  - Shell
compatibility: >-
  Router skill; requirements vary by sibling. Collectively they may need
  Linux x86_64, an NVIDIA GPU (Ampere+, CUDA 12.8, >= 24 GB VRAM) — not
  needed by five of the six `ncore-data-conversion` converters — Docker
  >= 23.0.1,
  NVIDIA Container Toolkit >= 1.13.5, an NGC API key, a Hugging Face
  token with the relevant gated licenses accepted, Python 3.10+, and
  `huggingface_hub`. Optional: CARLA / Isaac Sim 5.1 / AlpaSim for
  simulator integration over `serve-grpc`.
metadata:
  author: NVIDIA Physical AI
  tags:
    - physical-ai
    - nurec
    - neural-reconstruction
  upstream:
    repo: https://github.com/NVIDIA/nurec-skills
    branch: main
    skills_dir: skills/
    skills_dir_alias: .agents/skills/
    index_skill: skills/nurec-index/SKILL.md
    index_skill_name: nurec-index
    sibling_skills:
      - name: physical-ai-datasets
        folder: physical-ai-datasets/
        upstream: https://huggingface.co/nvidia
      - name: ncore-data-conversion
        former_name: ncore
        skill_repo: https://github.com/NVIDIA/ncore
        skill_path: skills/ncore-data-conversion/
        upstream: https://github.com/NVIDIA/ncore
      - name: nre
        folder: nre/
        upstream: nvcr.io/nvidia/nre/nre-ga
        tools_container: nvcr.io/nvidia/nre/nre-tools-ga
        release_tag: release_26.04
      - name: asset-harvester
        skill_repo: https://github.com/NVIDIA/asset-harvester
        skill_path: skills/asset-harvester/
        upstream: https://github.com/NVIDIA/asset-harvester
        hf_model: https://huggingface.co/nvidia/asset-harvester
      - name: harmonizer
        former_name: nurec-fixer
        skill_repo: https://github.com/NVIDIA/harmonizer
        skill_path: skills/harmonizer/
        upstream: https://github.com/NVIDIA/harmonizer
        hf_model: https://huggingface.co/nvidia/Harmonizer
        container: nvcr.io/nvidia/pytorch:25.10-py3
  upstream_clone_path: "${PHYSICAL_AI_SKILL_HUB_UPSTREAM_ROOT:-$HOME/.physical-ai-skill-hub/upstreams}/nurec-skills"
  upstream_override_env: NUREC_SKILLS_UPSTREAM_ROOT
---

# Physical AI Neural Reconstruction (NuRec) Router

## Purpose

This is a **thin router** for NVIDIA Neural Reconstruction (NuRec)
requests. It points at the upstream `nurec-index` skill at
`https://github.com/NVIDIA/nurec-skills` and its sibling skills
(`physical-ai-datasets`, `ncore-data-conversion`, `nre`, `asset-harvester`,
`harmonizer`). Use this skill to:

- Identify which upstream sibling skill answers a NuRec question.
- Locate, clone, or refresh the canonical `nurec-skills` checkout.
- Order multi-step NuRec workflows (data → conversion → train →
  render → cleanup) before opening the upstream recipe.

The canonical recipes (training, rendering, data conversion, dataset
downloads, object harvesting, frame cleanup) live in the upstream
sibling skills. **Never copy or reconstruct their commands here.**

**Do NOT use this skill for:**

- SimReady packaging of CAD or source meshes → use
  `omniverse-cad-to-simready`.
- Generic USD performance tuning unrelated to NuRec → use
  `omniverse-usd-performance-tuning`.
- AKS / OSMO / NIM Operator infrastructure setup → use
  `physical-ai-infrastructure-setup-and-resilient-scaling`.

## When to Use

Read this skill **first** whenever a user mentions any of:

`nurec`, `nurec router`, `nurec index`, `neural reconstruction`,
`neural reconstruction engine`, `NRE`, `3DGUT`, `3DGRT`, `USDZ`,
`NCore V4`, `sensorsim`, `sensor sim`, `novel view synthesis`,
`PhysicalAI-Autonomous-Vehicles-NuRec`, `PhysicalAI-Robotics-NuRec`,
`PhysicalAI-NuRec-PPISP`, `Cosmos-Drive-Dreams`, `asset harvester`,
`nurec fixer`, `DiffusionHarmonizer`, `harmonizer`, `difix`,
`difix3d`, `carline adaptation`, `serve-grpc`, `render-grpc`,
`warm serve-grpc`, `nre thin client`, `batch_render_rgb`,
`nurec teardown`, "where do I start with NuRec", "which NuRec skill
should I use for X?".

Decide which upstream sibling skill answers the question, fetch it
(see [Locate and fetch the upstream skills](#locate-and-fetch-the-upstream-skills)),
then follow that skill's body.

## Prerequisites

The router itself has no runtime prerequisites beyond `git` for
fetching the upstream. Downstream sibling skills require:

- **Linux x86_64** — aarch64 is not supported by `nre`.
- **NVIDIA GPU + driver** — CUDA 12.8 capability and >= 24 GB VRAM
  (48 GB+ recommended). Ampere (A100/A10/A40/RTX A6000), Ada
  (L20/L40/L40S), Hopper (H100/H20): R550+ required, R570+
  recommended. Blackwell (RTX Pro 6000D): R580+.
  `asset-harvester` needs driver >= 570 and ~16 GB VRAM. This applies to
  `nre`, `harmonizer`, `asset-harvester` and `ncore-data-conversion`'s
  **PAI** converter only — PAI decodes H.264 on the GPU, so its binary
  needs the stack even to start. The other five NCore converters need no
  GPU.
- **Docker >= 23.0.1 + NVIDIA Container Toolkit >= 1.13.5** — for the
  `nre`, `nre-tools`, and `harmonizer` containers
  (`nvcr.io/nvidia/nre/nre-ga:latest`,
  `nvcr.io/nvidia/nre/nre-tools-ga:latest`, and the locally-built
  `harmonizer-cosmos-env` image layered on
  `nvcr.io/nvidia/pytorch:25.10-py3`).
- **NGC API key** — for pulling `nvcr.io` containers. Resolution
  order is `$NGC_CLI_API_KEY` first, then `$NGC_API_KEY`, and only
  then prompt the user (see `nre`'s
  `references/ngc-and-registry.md`).
- **Hugging Face token** (`HF_TOKEN`) with the gated licenses
  **accepted in advance** on Hugging Face: `nvidia/PhysicalAI-*`
  datasets, `nvidia/Cosmos-Predict2-0.6B-Text2Image` and
  `nvidia/Harmonizer-Dataset`. The `nvidia/Harmonizer` checkpoints are
  public and download anonymously. The
  `nvidia/asset-harvester` checkpoints themselves are public; its
  optional DINOv3, Llama Guard and SAM 3D Body models are gated.
- **Python 3.10+** with `huggingface_hub` installed. For
  `ncore-data-conversion`, a checkout plus **Bazel via `bazelisk`** and a
  GitHub PAT with `read:packages` in `~/.netrc` — the `nvidia-ncore` wheel
  packages the library only and declares no console entry points, so it
  does not give you the converters. conda (Miniconda /
  Miniforge) for `asset-harvester`; it needs a GCC that `nvcc` accepts
  (10–13 is the tested range, but `setup.sh` selects its own compiler).
- **(Optional)** CARLA, Isaac Sim 5.1, or AlpaSim for simulator
  integration over `serve-grpc`.

Prefer each sibling's `scripts/validate_setup.py` (present in `nre`,
`asset-harvester`, and `harmonizer`) over hand-written checks.
`ncore-data-conversion` has none, and the checks below do not establish
its readiness — it needs a GitHub PAT with `read:packages` in
`~/.netrc` for Bazel, and `HF_TOKEN` only for the gated PAI converter;
follow its own Prerequisites. For `physical-ai-datasets` and this
router, verify secrets without echoing values:

```bash
hf auth whoami
[ -n "${HF_TOKEN:-}" ]         && echo "HF_TOKEN length=${#HF_TOKEN}"                 || echo "HF_TOKEN unset"
[ -n "${NGC_CLI_API_KEY:-}" ]  && echo "NGC_CLI_API_KEY length=${#NGC_CLI_API_KEY}"   || echo "NGC_CLI_API_KEY unset"
[ -n "${NGC_API_KEY:-}" ]      && echo "NGC_API_KEY length=${#NGC_API_KEY}"           || echo "NGC_API_KEY unset"
```

See [`references/secrets-handling.md`](references/secrets-handling.md)
for the bash anti-patterns to avoid.

## What is NuRec?

**NuRec** (NVIDIA Omniverse Neural Reconstruction) takes camera,
LiDAR, radar, or stereo recordings — typically from a self-driving car
or a robot — and turns them into a 3D scene you can re-render from any
viewpoint. Names that come up a lot:

- **NRE** — "Neural Reconstruction Engine". NuRec is the product; NRE
  is the engine that trains and renders. Both route to the upstream
  `nre` skill.
- **USDZ** — the file format of a trained scene. A zip archive that
  Omniverse, Isaac Sim, and CARLA can open.
- **NCore V4** — the input format NRE consumes. Raw recordings must be
  converted to NCore V4 before training.
- **3DGUT / 3DGRT** — the two 3D Gaussian Splatting flavours used
  internally by NRE. The default Hydra recipe picks one; most users
  never set it manually.

A typical NuRec project has three stages:

1. **Get the input** — convert your own recording to NCore V4
   (`ncore-data-conversion`), or download a pre-converted dataset
   (`physical-ai-datasets`).
2. **Train the reconstruction** — feed NCore V4 to NRE; out comes a
   USDZ (`nre`).
3. **Render new views** — render images, videos, or LiDAR sweeps from
   the USDZ (`nre`).

Projects that just want to *use* an existing NVIDIA-published scene
skip step 2.

## Pick a skill

Match the user's goal in the left column and open the named upstream
skill on the right. Arrows mean "do these in order".

| I want to… | Upstream skill |
|------------|----------------|
| Find or download a NuRec dataset NVIDIA has published | `physical-ai-datasets` |
| Convert my own camera / LiDAR / radar / depth / stereo recording into NCore V4 | `ncore-data-conversion` |
| Adapt the nearest converter for an unsupported sensor setup (drone, RGB-D, ROS 2 bag) | `ncore-data-conversion` |
| Train a 3D reconstruction from an NCore clip | `nre` (the clip is already V4 — no conversion needed; add `ncore-data-conversion` only to inspect or diagnose it) |
| Generate the extra inputs NRE needs (segmentation masks, depth, ego mask, DINOv2, LiDAR-seg visibility) | `nre` (uses the `nre-tools-ga` container) |
| Render a USDZ along the original camera positions | `nre` |
| Render at full resolution / highest quality | `nre` (see "Quality presets") |
| Render along a shifted trajectory (e.g. car moved 3 m left) | `nre` |
| Adapt an existing USDZ to an augmented target-vehicle rig (carline adaptation) | `nre` (`export-custom-rig-trajectory` → `render`) → `harmonizer` |
| Render through a server so CARLA / Isaac Sim / AlpaSim / a custom simulator can ask for frames | `nre` (`serve-grpc`) |
| Render the same USDZ many times back-to-back from Python with minimal per-call latency | `nre` (warm `serve-grpc` + thin Python client / `batch_render_rgb`) |
| Render LiDAR sweeps (point clouds) from a USDZ | `nre` (`render-grpc --lidar`) |
| Skip training and just render a NuRec scene NVIDIA already built | `physical-ai-datasets` → `nre` |
| Skip training and use a pre-built indoor robotics scene | `physical-ai-datasets` → `nre` (then Isaac Sim 5.1) |
| Extract individual 3D objects (cars, pedestrians) from a driving clip | `asset-harvester` |
| Add, remove, or replace cars / pedestrians in a NuRec scene | `asset-harvester` → `nre` |
| Clean up or harmonize rendered frames (ghosting, floaters, flicker, lighting/shadows) | `harmonizer`, **or** `--enable-difix` inside `nre` for inline rendering |
| Export the scene as a PLY, mesh, depth maps, ego mask, etc. | `nre` |
| Upgrade an old USDZ so newer NRE versions load it faster | `nre` (`upgrade-artifact`) |
| Open a USDZ or PLY in a browser viewer | `nre` (`viewer` / `ply_viewer`) |
| Measure rendering quality (PSNR, SSIM, LPIPS) against ground truth | `nre` (`eval-rendering-metrics`) |
| Benchmark different reconstruction methods on the same scenes | `physical-ai-datasets` (`PhysicalAI-NuRec-PPISP`) → `nre` |
| Train on multiple GPUs or on SLURM | `nre` |

## Common workflows

Seven end-to-end workflows are documented in
[`references/workflows.md`](references/workflows.md), lettered to
match the upstream `nurec-index` workflow IDs:

- **A.** Make a NuRec scene from your own recording.
- **B.** Use a NuRec scene NVIDIA has already trained.
- **C.** Use NuRec for indoor robot simulation.
- **D.** Add, remove, or replace 3D objects in a scene.
- **E.** Clean up rendered frames.
- **F.** Benchmark reconstruction quality.
- **G.** Connect NuRec to a simulator.

Open that file when the user's task spans more than one sibling skill.

## Sibling skills (upstream)

Refer to a sibling by its **name** — that is the portable identifier.
The folder column is where it lives in a local `nurec-skills` checkout,
except where a repo is named — `asset-harvester`, `harmonizer` and
`ncore-data-conversion` ship from their own product repos.

| Name | Upstream folder | What it does |
|------|-----------------|--------------|
| `physical-ai-datasets` | `skills/physical-ai-datasets/` | Catalog and download recipes for every NVIDIA Physical AI dataset on Hugging Face (driving, robotics, manipulation, NuRec scenes, benchmarks). |
| `ncore-data-conversion` | [`NVIDIA/ncore`](https://github.com/NVIDIA/ncore) → `skills/ncore-data-conversion/` | Converts any sensor recording to NCore V4 (the format NRE needs). Also covers adapting the nearest built-in converter. |
| `nre` | `skills/nre/` | The Neural Reconstruction Engine itself (`nvcr.io/nvidia/nre/nre-ga`, `nvcr.io/nvidia/nre/nre-tools-ga`, NRE `release_26.04`). Trains, performs carline adaptation, renders (locally, via warm `serve-grpc` + thin Python client / `batch_render_rgb`, or to an external simulator), exports meshes / point clouds / depth, edits actors, evaluates quality. |
| `asset-harvester` | [`NVIDIA/asset-harvester`](https://github.com/NVIDIA/asset-harvester) → `skills/asset-harvester/` | Open-source Apache-2.0 pipeline (SparseViewDiT + TokenGS) that extracts individual 3D objects from sparse views in a driving clip and saves them as `.ply` Gaussian splats, optionally emitting `metadata.yaml` for the NuRec handoff. |
| `harmonizer` | [`NVIDIA/harmonizer`](https://github.com/NVIDIA/harmonizer) → `skills/harmonizer/` | Standalone NVIDIA **DiffusionHarmonizer** workflow — public successor to the older Fixer / Difix3D+ recipes — that cleans rendered frames, harmonizes inserted actors, evaluates PSNR/LPIPS, and optionally fine-tunes the model. |

For naming overlaps (NRE vs Fixer, ncore-data-conversion vs nre, AV-NuRec vs
Cosmos-Drive-Dreams, NuRec vs SimReady) see
[`references/mix-ups.md`](references/mix-ups.md).

## Locate and fetch the upstream skills

Try the local disk first, in this order — a sibling skill already
installed in the runtime is preferable to a network fetch. `harmonizer` and
`ncore-data-conversion` participate too: both were renamed, so a hit under
either new name cannot be a stale copy — those are still called
`nurec-fixer` and `ncore`. `asset-harvester` does not, because its name
did not change (see `references/upstream-fetch.md`):

1. `.agents/skills/<name>/SKILL.md` (Cursor, Codex, NemoClaw)
2. `.claude/skills/<name>/SKILL.md` (Claude Code)
3. `.cursor/skills/<name>/SKILL.md` (project-scoped)
4. `~/.cursor/skills/<name>/SKILL.md` (personal skills)
5. An existing `nurec-skills` clone under the shared upstream root.

**Never fall back to `nurec-fixer` or `ncore`** — those are the stale
pre-rename copies.
`asset-harvester` is excluded from this order entirely; see
[`references/upstream-fetch.md`](references/upstream-fetch.md).

**Only if none of those exist**, ask the user for explicit consent
before cloning. A `git clone` is a network fetch of an external
repository plus a write to the local filesystem; it can violate
org network policy and carries supply-chain risk. Show the user
what you intend to run and wait for a yes.

The recipe below clones `nurec-skills`, so it serves **only the two
siblings hosted there**. A missing `harmonizer` comes from
`NVIDIA/harmonizer`, `ncore-data-conversion` from `NVIDIA/ncore`, and
`asset-harvester` from `NVIDIA/asset-harvester` — never from
`nurec-skills`, which holds only their stale pre-move copies. See
[`references/upstream-fetch.md`](references/upstream-fetch.md).

Quick recipe (full version, including the pinned-commit layout, in
[`references/upstream-fetch.md`](references/upstream-fetch.md)):

```bash
UPSTREAM_ROOT="${NUREC_SKILLS_UPSTREAM_ROOT:-${PHYSICAL_AI_SKILL_HUB_UPSTREAM_ROOT:-$HOME/.physical-ai-skill-hub/upstreams}}"
mkdir -p "$UPSTREAM_ROOT"
if [ -d "$UPSTREAM_ROOT/nurec-skills/.git" ]; then
  git -C "$UPSTREAM_ROOT/nurec-skills" fetch --tags
  git -C "$UPSTREAM_ROOT/nurec-skills" checkout main
  git -C "$UPSTREAM_ROOT/nurec-skills" pull --ff-only
else
  # Only after the user has agreed. Prefer --branch <tag-or-sha> over HEAD.
  git clone --depth 1 https://github.com/NVIDIA/nurec-skills.git \
    "$UPSTREAM_ROOT/nurec-skills"
fi
test -f "$UPSTREAM_ROOT/nurec-skills/skills/nurec-index/SKILL.md"
```

The upstream tree is rooted at `skills/<name>/SKILL.md`;
`.agents/skills` is a symlink onto `skills/`, so either path
resolves. Read the upstream skill before running any mutating
command:

```bash
cat "$UPSTREAM_ROOT/nurec-skills/skills/nurec-index/SKILL.md"  # upstream router
cat "$UPSTREAM_ROOT/nurec-skills/skills/<folder>/SKILL.md"     # sibling
```

Companion files (`references/`, `scripts/`, `assets/`) live next to
**the sibling's** `SKILL.md`, not next to this router.

## Hard Rules

- Router only — do not duplicate upstream NuRec recipes here. Read
  the upstream sibling skill body before running any mutating command.
- Refer to sibling skills by their `name:` (e.g. `nre`), not by repo
  path. Folder layouts can change; the name is portable.
- **Never `git clone` the upstream without explicit user consent.**
  For the `nurec-skills` siblings, exhaust the local lookup order
  first, show the exact command, and clone only into a path the user
  agreed to — never silently into
  `/tmp`. Do not scan broad developer workspaces such as `~/Codes` or
  reuse unrelated old clones.
- Use the GA container channel: `nvcr.io/nvidia/nre/nre-ga` and
  `nvcr.io/nvidia/nre/nre-tools-ga`. The un-suffixed
  `nvcr.io/nvidia/nre/nre` / `nre-tools` names are the legacy
  channel — still valid for cached version pins, but not what a new
  workflow should pull.
- Resolve the NGC key as `${NGC_CLI_API_KEY:-${NGC_API_KEY:-}}` and
  log in with `docker login nvcr.io --username '$oauthtoken'
  --password-stdin`. Never echo a key.
- `physical-ai-datasets` covers gated Hugging Face datasets. Do not
  bypass dataset license terms; the user must accept the
  `PhysicalAI-*` gated licenses on Hugging Face and provide a token
  before downloading.
- Asset Harvester runs **before** packaging into a USDZ. Do not call
  `nre`'s `export-external-assets` on hand-rolled `.ply` files unless
  the user explicitly asks to skip Asset Harvester.
- For artifact cleanup, prefer the built-in `--enable-difix` path in
  `nre`. Route to the standalone `harmonizer` only when the user
  needs the public code/model card, paired evaluation, fine-tuning,
  or fixes on previously rendered frames.
- Do not invent NRE / NCore / DiffusionHarmonizer commands from
  memory. Re-read the upstream sibling skill — versions move fast
  (NRE `release_26.04` is the current pin. NCore ships semver tags, but the
  `ncore-data-conversion` skill is newer than the latest of them — take it
  from `main`, not from a release tag.)
- This router does not deploy infrastructure. Route AKS / OSMO /
  NIM Operator setup to
  `physical-ai-infrastructure-setup-and-resilient-scaling`.

## Limitations

- **Router only.** This skill never executes mutating NuRec commands.
  All training, rendering, conversion, and harmonization happens in
  upstream sibling skills.
- **Upstream version drift.** Most recipes live in
  `https://github.com/NVIDIA/nurec-skills`; `asset-harvester`,
  `harmonizer` and `ncore-data-conversion` live in their own product
  repos, which evolve outside
  this repo. Stale clones can drift; always refresh the upstream
  before relying on a sibling skill.
- **Hand-curated catalogue.** A newly-added upstream sibling is not
  discoverable here until someone edits the tables (see
  [`references/maintenance.md`](references/maintenance.md)).
- **Gated content.** `nvidia/PhysicalAI-*`,
  `nvidia/Cosmos-Predict2-0.6B-Text2Image` and `nvidia/Harmonizer-Dataset`
  require the user to accept license terms on Hugging Face first.
  `nvidia/Harmonizer` itself is public. For `asset-harvester` only its
  optional DINOv3, Llama Guard and SAM 3D Body models are gated.
  The router cannot bypass this.
- **Heavy footprint.** A complete NuRec workflow can leave 150 GB+
  on disk. See [`references/teardown.md`](references/teardown.md).
- **NVIDIA-only stack.** Requires Linux x86_64. Most of the workflow
  also needs an NVIDIA GPU and the NVIDIA Container Toolkit — the
  exception is `ncore-data-conversion`, where only the PAI converter
  needs the GPU stack. aarch64 / AMD / Intel / Apple Silicon are not
  supported.
- **No Omniverse / Isaac Sim integration steps.** Handing a USDZ to
  Isaac Sim 5.1 (workflow C) is documented in the Isaac Sim docs, not
  in the NuRec skill family.
- **Not a SimReady pipeline.** NuRec produces a renderable USDZ from
  a recording; SimReady packaging of CAD or source meshes is a
  different pipeline (see `omniverse-cad-to-simready`).

## Troubleshooting

| Error / symptom | Likely cause | Solution |
|-----------------|--------------|----------|
| `nurec-skills` clone missing or empty | Upstream not fetched yet | Walk the local lookup order, then ask consent and run the clone block in [Locate and fetch the upstream skills](#locate-and-fetch-the-upstream-skills) |
| `test -f .../.agents/skills/SKILL.md` fails | Wrong upstream path — the index lives at `skills/nurec-index/SKILL.md` | Use `skills/nurec-index/SKILL.md` (or the `.agents/skills/` symlink alias) |
| `403`/`401` pulling `nvidia/PhysicalAI-*` from HF | Gated license not accepted, or `HF_TOKEN` unset / wrong scope | Accept the gated license on Hugging Face, then `hf auth login` with a token that has `read` access |
| `denied: requested access to the resource is denied` from `nvcr.io/nvidia/nre/*` | Missing or expired NGC key | `docker login nvcr.io` with `$oauthtoken` / `${NGC_CLI_API_KEY:-$NGC_API_KEY}`; rotate at `org.ngc.nvidia.com/setup/api-key` if needed |
| `manifest unknown` / `not found` pulling an NRE image | Pulling the legacy un-suffixed name or a tag that channel never published | Pull the GA names `nvcr.io/nvidia/nre/nre-ga:latest` and `nvcr.io/nvidia/nre/nre-tools-ga:latest` |
| `--renderer` or `export-custom-rig-trajectory` rejected as unknown | Cached image is older than `26.04` / `26.03` | Pull a `26.04+` GA image; `--image-format jpeg` works on every family, so don't fall back to PNG |
| NRE refuses to load a clip ("not valid NCore V4") | Recording was not converted | Run the `ncore-data-conversion` skill before invoking `nre` |
| `serve-grpc` cold-start latency dominates a Python loop | One-shot Docker invocation per render | Use the `nre` warm `serve-grpc` + thin Python client (`batch_render_rgb`) recipe; the warm fast path needs a `26.04+` image |
| Output files are owned by `root` after a `docker run` | `-u $(id -u):$(id -g)` was missing | `sudo chown -R "$(id -u):$(id -g)" <output_dir>`; add the `-u` flag next time |
| Frames have ghosting / floaters / flicker after rendering | Inline cleanup not enabled | Re-render with `nre --enable-difix`, or post-process with `harmonizer` (DiffusionHarmonizer) |
| Stale **skill** names (`nurec-fixer`, `ncore` as a sibling identifier or `skills/ncore/` route), `nvidia/Fixer`, `nvidia/DiffusionHarmonizer` weights in agent output | Out-of-date cached skill | The skills are now `harmonizer` and `ncore-data-conversion`. Bare **NCore** remains correct for the product, repo, `nvidia-ncore` library and `ncore_vis` tool; the model now lives at `nvidia/Harmonizer` — see [`references/maintenance.md`](references/maintenance.md) |
| Bash anti-pattern `${HF_TOKEN:+yes}${HF_TOKEN:-no}` echoed token value | Misuse of bash parameter expansion | Rotate the token; use `hf auth whoami` or length-only checks (see [`references/secrets-handling.md`](references/secrets-handling.md)) |

## Cross-skill teardown

A complete NuRec workflow can leave **150 GB+** on disk between
container images, model weights, code clones, conda envs, and output
directories. Most sibling skills have their own dedicated `Teardown`
section — read them in the order documented in
[`references/teardown.md`](references/teardown.md) when the user no
longer needs the workflow. Do **not** revoke `NGC_API_KEY` /
`HF_TOKEN` as part of teardown unless they were leaked.

## Keeping this router up to date

Procedure for adding new sibling skills, renames, or upstream URL
changes lives in [`references/maintenance.md`](references/maintenance.md).
Treat the upstream `nurec-index` at
<https://github.com/NVIDIA/nurec-skills/blob/main/skills/nurec-index/SKILL.md>
as authoritative **for the routing taxonomy and workflow ordering**;
this skill mirrors only the picker tables, the workflow ordering, and
the upstream fetch recipe. It is not authoritative for
`asset-harvester`, `harmonizer` or `ncore-data-conversion`, which are
maintained in their own product repos.
