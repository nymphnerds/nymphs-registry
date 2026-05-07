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
github.com/nymphnerds/ai-toolkit
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
```

For complex modules like AI Toolkit or Brain, the module page can use many more commands later.

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
