# Cross-Skill Teardown

A complete NuRec workflow can leave **150 GB+ on disk** between
container images, model weights, code clones, conda envs, and output
directories. Each sibling skill has its own dedicated "Teardown"
section — read them in this order when the user no longer needs the
workflow:

| Sibling skill | Approximate footprint | Where the cleanup lives |
|---------------|------------------------|--------------------------|
| `nre` | ~120 GB images + caches + per-run outputs | `nre/SKILL.md#teardown` + `nre/references/teardown.md` |
| `harmonizer` | ~120 GB free for inference; the optional training dataset is a separate 1.76 TB download needing further extraction headroom | [`skills/harmonizer/references/teardown.md#reclaim-disk`](https://github.com/NVIDIA/harmonizer/blob/main/skills/harmonizer/references/teardown.md#reclaim-disk) |
| `asset-harvester` | ~60 GB+ — checkpoints (~12.8 GB), two conda envs (~10–15 GB each), benchmark assets, outputs | [`skills/asset-harvester/references/troubleshooting.md#teardown`](https://github.com/NVIDIA/asset-harvester/blob/main/skills/asset-harvester/references/troubleshooting.md#teardown) |
| `ncore` | clip-dependent | NCore shards live under `<dataset_dir>/`; delete after `nre` training is done |
| `physical-ai-datasets` | dataset-dependent | HF caches under `${HF_HOME:-$HOME/.cache/huggingface}/hub/`; remove the per-dataset directory |

Two practical rules that apply across every container-based sibling:

1. Pin `-u $(id -u):$(id -g)` on every `docker run` so outputs land
   owned by the user, not by `root`. The canonical `nre` and
   `harmonizer` runs do this, but not every documented example does —
   `harmonizer`'s interactive inference command omits it — so check the
   command you are about to run. If outputs end up
   `root`-owned anyway, follow the chosen sibling's own teardown
   recovery rather than a broad `chown -R`, which would also seize
   files belonging to collaborators.
2. Do **not** revoke `NGC_CLI_API_KEY` / `NGC_API_KEY` / `HF_TOKEN`
   as part of teardown unless there is reason to believe they were
   leaked — they are per-user and shared across every NVIDIA workflow
   on the host.
