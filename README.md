# Nymphs Registry

This repo is the trusted module catalog for **NymphsCore Manager**.

It tells the Manager which Nymph modules exist, which ones are trusted, and where to find each module's `nymph.json`.

It does **not** contain module payloads.

Plain English:

```text
nymphs-registry = catalog
module repo     = actual module
```

Current registry file:

```text
nymphs.json
```

Current first test module:

```text
github.com/nymphnerds/worbi
```

---

## What Will Work

The intended Manager flow is:

1. NymphsCore Manager fetches this registry.
2. Manager reads `nymphs.json`.
3. Manager sees official/trusted modules.
4. Manager follows each `manifest_url`.
5. Manager reads the module's `nymph.json`.
6. Manager shows the module as available or installed.
7. Manager uses the module's stable wrapper scripts for install/status/start/stop/open/logs.

For WORBI, this means:

```text
registry
  -> https://raw.githubusercontent.com/nymphnerds/worbi/main/nymph.json
  -> WORBI manifest
  -> WORBI wrapper scripts
  -> install/start/open/stop/status/logs
```

The Manager should not search all of GitHub or the internet. It should only trust modules listed here.

---

## Current WORBI Test

WORBI is the first registry test.

WORBI repo:

```text
https://github.com/nymphnerds/worbi
```

WORBI manifest:

```text
https://raw.githubusercontent.com/nymphnerds/worbi/main/nymph.json
```

WORBI is currently treated as a simple launcher/status module:

- install or repair WORBI
- start WORBI
- open WORBI
- stop WORBI
- show status
- show logs

WORBI itself owns its deeper app UI.

Default WORBI runtime:

```text
install root: ~/worbi
frontend:     http://localhost:5173
backend:      http://localhost:8082
health:       http://localhost:8082/api/health
logs:         ~/worbi/logs
```

---

## Registry Shape

`nymphs.json` uses this shape:

```json
{
  "registry_version": 1,
  "updated": "2026-05-07",
  "modules": [
    {
      "id": "worbi",
      "name": "WORBI",
      "short_name": "WB",
      "category": "tool",
      "packaging": "archive",
      "summary": "Local-first worldbuilding app for projects, lore, characters, and creative planning.",
      "install_root": "$HOME/worbi",
      "sort_order": 50,
      "channel": "test",
      "trusted": true,
      "manifest_url": "https://raw.githubusercontent.com/nymphnerds/worbi/main/nymph.json"
    }
  ]
}
```

Field meanings:

- `registry_version`: version of this registry format
- `updated`: date the catalog was last intentionally changed
- `modules`: list of trusted modules
- `id`: stable module id used by the Manager
- `name`: display name
- `short_name`: compact display label for navigation/cards
- `category`: broad UI grouping, such as `service`, `image`, `training`, `3d`, or `tool`
- `packaging`: module source/package type, such as `repo` or `archive`
- `summary`: short card description that can be shown before install
- `install_root`: expected install root inside the managed Linux user home
- `sort_order`: stable card/navigation ordering
- `channel`: release lane such as `test`, `stable`, `experimental`
- `trusted`: whether this entry is trusted by NymphsCore
- `manifest_url`: raw URL to the module repo's `nymph.json`

---

## Module Repo Rule

Use one clean repo per Nymph module.

Examples:

```text
github.com/nymphnerds/worbi
github.com/nymphnerds/zimage
github.com/nymphnerds/trellis
github.com/nymphnerds/brain
github.com/nymphnerds/lora
```

Each module repo should own:

```text
nymph.json
README.md
scripts/
packages/ or source files as needed
```

The module repo can clone or depend on public upstream repos internally, but the Manager should only need:

```text
registry -> module manifest -> stable wrapper scripts
```

---

## Manifest Rule

Each module has one `nymph.json`.

The manifest describes static module information:

- id
- name
- version
- kind
- category
- source
- entrypoints
- capabilities
- basic runtime URLs/paths

The manifest should **not** store live state.

Do not put these in `nymph.json`:

- installed/not installed
- running/stopped
- outdated/current
- PID values
- live health result

The Manager detects live state separately by calling the module's `status` entrypoint.

---

## Script Contract

The Manager should call wrapper scripts from the module repo.

Common entrypoints:

```text
install
status
start
stop
open
logs
uninstall
update
configure
```

Not every module needs every entrypoint.

For simple launcher modules like WORBI, this is enough:

```text
install
status
start
stop
open
logs
uninstall
```

For complex modules like AI Toolkit or Brain, the module page can use many more commands later.

### Uninstall Contract

Every official module should provide an uninstall entrypoint once it is installable:

```text
scripts/<module>_uninstall.sh
```

The uninstall script must support:

```text
--dry-run   print exactly what would be deleted and preserved
--yes       required before actual deletion
--purge     delete everything for that module, including user data
```

Default uninstall should remove the installed runtime/program files and return the module to an available/not-installed state, while preserving user data where that matters.

Examples of data to preserve by default:

```text
Z-Image     -> outputs, logs
TRELLIS     -> outputs, logs
Brain       -> models, Open WebUI data, MCP config/data, secrets, logs
LoRA        -> datasets, jobs, generated LoRAs, config, logs
WORBI       -> data, projects, config, logs
```

`--purge` is the explicit "delete everything" mode. The Manager should only call it after showing the exact delete list and getting a stronger confirmation from the user.

The manifest should expose the uninstall entrypoint and policy:

```json
{
  "manager": {
    "uninstall": "scripts/lora_uninstall.sh"
  },
  "uninstall": {
    "supports_purge": true,
    "requires_confirmation": true,
    "dry_run_arg": "--dry-run",
    "confirm_arg": "--yes",
    "purge_arg": "--purge",
    "preserve_by_default": ["datasets", "loras", "jobs", "config", "logs"]
  }
}
```

---

## Manager Page Rule

The registry does not force every module to have the same UI.

Each module can be completely different:

```text
Brain       -> local LLM, MCP, WebUI, model manager, OpenRouter, monitor
Z-Image     -> image runtime readiness, model fetch, smoke test
TRELLIS     -> 3D runtime readiness, GGUF quant, model fetch, adapter repair
AI Toolkit  -> datasets, captions, LoRAs, jobs, queue, progress
WORBI       -> launcher/status page for the local worldbuilding app
```

The shared shell is only:

```text
Home cards
Installed/available state
Navigation
Registry discovery
Trusted manifest loading
```

The content page should be owned by the module.

---

## Security Rules

The Manager should be conservative.

It should:

- fetch this trusted registry
- read only manifests linked by this registry
- validate basic manifest fields
- show source repo clearly
- call stable wrapper scripts
- keep live status separate from static manifests

It should not:

- search random GitHub repos
- run arbitrary scripts from untrusted URLs
- treat live state as manifest data
- require every module to share one generic UI page

---

## Adding A New Module

To add a module:

1. Create a module repo under `github.com/nymphnerds/`.
2. Add a root `nymph.json`.
3. Add stable wrapper scripts.
4. Test the scripts manually.
5. Add an entry to `nymphs.json`.
6. Commit and push this registry.

Example entry:

```json
{
  "id": "example",
  "name": "Example",
  "channel": "test",
  "trusted": true,
  "manifest_url": "https://raw.githubusercontent.com/nymphnerds/example/main/nymph.json"
}
```

---

## Planned Modules

Current planned module set:

```text
worbi
zimage
trellis
brain
ai-toolkit
```

WORBI is first because it is the cleanest external module test.

The others should be migrated carefully from the existing NymphsCore Manager scripts and existing installed backend folders.

---

## Guidance For The Other Modules

The long-term rule stays the same:

```text
one clean repo per module
one nymph.json per module
one registry entry per module
module-specific Manager page content
```

The registry should eventually list every official module, but each module should be prepared in its own repo first.

### Z-Image Turbo

Planned repo:

```text
github.com/nymphnerds/zimage
```

Expected install root:

```text
~/Z-Image
```

Category:

```text
runtime
```

Purpose:

```text
Z-Image Turbo image generation runtime.
```

The module repo should own:

```text
nymph.json
README.md
scripts/install_zimage.sh
scripts/zimage_status.sh
scripts/zimage_start.sh
scripts/zimage_stop.sh
scripts/zimage_open.sh
scripts/zimage_logs.sh
scripts/zimage_fetch_models.sh
scripts/zimage_smoke_test.sh
```

Manager page should focus on:

- runtime readiness
- model availability
- Nunchaku/diffusers/runtime state
- model fetch
- smoke test
- server URL
- logs

Existing NymphsCore sources to migrate from:

```text
Manager/scripts/install_nymphs2d2.sh
Manager/scripts/zimage_backend_overlay/
Manager/scripts/runtime_tools_status.sh
Manager/scripts/smoke_test_server.sh
Manager/scripts/prefetch_models.sh
```

Example future registry entry:

```json
{
  "id": "zimage",
  "name": "Z-Image Turbo",
  "channel": "experimental",
  "trusted": true,
  "manifest_url": "https://raw.githubusercontent.com/nymphnerds/zimage/main/nymph.json"
}
```

### TRELLIS.2

Planned repo:

```text
github.com/nymphnerds/trellis
```

Expected install root:

```text
~/TRELLIS.2
```

Category:

```text
runtime
```

Purpose:

```text
TRELLIS.2 3D generation runtime with Nymphs GGUF adapter support.
```

The module repo should own:

```text
nymph.json
README.md
scripts/install_trellis.sh
scripts/trellis_status.sh
scripts/trellis_start.sh
scripts/trellis_stop.sh
scripts/trellis_open.sh
scripts/trellis_logs.sh
scripts/trellis_fetch_models.sh
scripts/trellis_smoke_test.sh
scripts/trellis_repair_adapter.sh
```

Manager page should focus on:

- TRELLIS readiness
- GGUF quant selection
- support model availability
- adapter repair
- model fetch
- smoke test
- server URL
- logs

Existing NymphsCore sources to migrate from:

```text
Manager/scripts/install_trellis.sh
Manager/scripts/trellis_adapter/
Manager/scripts/runtime_tools_status.sh
Manager/scripts/smoke_test_server.sh
Manager/scripts/prefetch_models.sh
```

TRELLIS may internally depend on upstream repos such as:

```text
microsoft/TRELLIS.2
Aero-Ex/ComfyUI-Trellis2-GGUF
city96/ComfyUI-GGUF
```

But the Manager should only see the Nymphs module repo and its manifest.

Example future registry entry:

```json
{
  "id": "trellis",
  "name": "TRELLIS.2",
  "channel": "experimental",
  "trusted": true,
  "manifest_url": "https://raw.githubusercontent.com/nymphnerds/trellis/main/nymph.json"
}
```

### Brain

Planned repo:

```text
github.com/nymphnerds/brain
```

Expected install root:

```text
~/Nymphs-Brain
```

Category:

```text
service
```

Purpose:

```text
Local coding/model orchestration stack: local LLM, MCP, WebUI, model manager, and optional remote router support.
```

The module repo should own source/wrapper files, not the installed runtime output.

Good repo shape:

```text
nymph.json
README.md
scripts/install_brain.sh
scripts/brain_status.sh
scripts/brain_start.sh
scripts/brain_stop.sh
scripts/brain_open.sh
scripts/brain_logs.sh
scripts/brain_update.sh
templates/
```

Do not commit runtime output:

```text
venv/
mcp-venv/
open-webui-venv/
models/
open-webui-data/
secrets/
logs/
```

Manager page should preserve the serious page work from main:

- LLM status
- MCP status
- WebUI status
- local model
- remote model
- OpenRouter key state
- live monitor
- context/VRAM/tokens
- start/stop
- open WebUI
- manage models
- update stack
- logs

Existing NymphsCore sources to migrate from:

```text
Manager/scripts/install_nymphs_brain.sh
Manager/scripts/monitor_query.sh
Manager/scripts/remote_llm_mcp/
Manager/apps/NymphsCoreManager main-branch Brain page/viewmodel/service methods
```

Example future registry entry:

```json
{
  "id": "brain",
  "name": "Brain",
  "channel": "experimental",
  "trusted": true,
  "manifest_url": "https://raw.githubusercontent.com/nymphnerds/brain/main/nymph.json"
}
```

### AI Toolkit

Planned repo:

```text
github.com/nymphnerds/ai-toolkit
```

Expected install root:

```text
~/ZImage-Trainer
```

Current reference install:

```text
live WSL distro: NymphsCore
path:            /home/nymph/ZImage-Trainer
```

That live install is working and should be used as the practical reference when designing the module repo and Manager page.

Category:

```text
trainer
```

Purpose:

```text
AI Toolkit sidecar for Z-Image Turbo LoRA training.
```

This module can internally clone or wrap upstream:

```text
ostris/ai-toolkit
```

The live reference checkout currently uses:

```text
path:   /home/nymph/ZImage-Trainer/ai-toolkit
origin: https://github.com/ostris/ai-toolkit.git
branch: main
```

But the Manager should only see:

```text
github.com/nymphnerds/ai-toolkit
```

The module repo should own:

```text
nymph.json
README.md
scripts/install_ai_toolkit.sh
scripts/ai_toolkit_status.sh
scripts/ai_toolkit_start.sh
scripts/ai_toolkit_stop.sh
scripts/ai_toolkit_open.sh
scripts/ai_toolkit_logs.sh
scripts/ai_toolkit_create_job.sh
scripts/ai_toolkit_start_job.sh
scripts/ai_toolkit_stop_job.sh
scripts/ai_toolkit_caption_with_brain.sh
templates/
adapters/
```

Do not commit user/runtime output:

```text
datasets/
loras/
jobs/
logs/
models/
.node20/
.venv-ztrain/
ai-toolkit/venv/
ai-toolkit/node_modules/
ai-toolkit/ui/node_modules/
ai-toolkit/aitk_db.db
```

The live reference install already contains real datasets, LoRAs, jobs, and models, so the first migration pass should copy only source/wrapper/config pieces, never the training output.

Manager page should preserve the serious Z-Image Trainer work from main:

- datasets
- LoRAs
- captions
- Brain caption helper
- presets
- adapter version
- sample prompt
- steps
- checkpoints
- learning rate
- rank
- low VRAM mode
- add/start/stop/delete job
- queue state
- progress
- logs

Existing NymphsCore sources to migrate from:

```text
Manager/scripts/install_zimage_trainer_aitk.sh
Manager/scripts/zimage_trainer_status.sh
Manager/scripts/ztrain_run_config.sh
Manager/scripts/zimage_caption_brain.sh
Manager/scripts/zimage_caption_brain.py
Manager/scripts/compare_zimage_loras.py
Manager/apps/NymphsCoreManager main-branch Z-Image Trainer page/viewmodel/service methods
```

Example future registry entry:

```json
{
  "id": "ai-toolkit",
  "name": "AI Toolkit",
  "channel": "experimental",
  "trusted": true,
  "manifest_url": "https://raw.githubusercontent.com/nymphnerds/ai-toolkit/main/nymph.json"
}
```

---

## When To Add A Module To This Registry

Only add a module to `nymphs.json` when:

- the module repo exists
- the module has a root `nymph.json`
- the manifest URL works as a raw GitHub URL
- the status script exits `0` even when not installed
- install/start/stop/open/logs scripts are at least safe stubs
- the Manager can show the module as available without crashing

For early work, use:

```json
"channel": "test"
```

For modules that are usable but still evolving, use:

```json
"channel": "experimental"
```

Reserve:

```json
"channel": "stable"
```

for modules that are safe for normal users.
