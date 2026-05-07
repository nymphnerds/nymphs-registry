# Nymphs Registry Versioning System — Analysis & Implementation Handoff

**Date:** 2026-05-07
**Author:** Cline (AI Analysis)
**Status:** Proposal / Ready for Implementation

---

## Executive Summary

The `nymphs-registry/nymphs.json` file already contains a `registry_version` field (currently set to `1`), but **NymphsCore Manager does not currently consume this registry file at all**. This document outlines the current state, why a registry-based update system should be implemented, and what is required on both the registry and Manager sides.

---

## 1. Current State

### 1.1 Registry File (`nymphs-registry/nymphs.json`)

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
    },
    // ... 4 more modules
  ]
}
```

The registry serves as a **central index** of available Nymphs modules. Each entry points to a remote `nymph.json` manifest hosted on the module's GitHub repository.

### 1.2 Manager Update Detection (Today)

Manager's existing update flow is **git-based and WSL-internal**:

- `RunManagedRepoUpdateCheckAsync()` calls into the WSL distro and runs `git fetch` + divergence checks on repos already cloned inside WSL (Z-Image, TRELLIS.2, NymphsCore helper repo).
- The result is parsed from structured `repo=...|state=...` lines into states like `up_to_date`, `behind_clean`, `dirty`, `branch_mismatch`, etc.
- `addons.json` is **bundled with the Manager binary** at build time and never changes without a new Manager release.

**Critical finding:** The registry file is never fetched, parsed, or consulted by Manager. It exists as a GitHub resource but has no consumer.

### 1.3 Manager Addon Metadata (`addons.json`)

`addons.json` is a local, bundled file that defines 5 addons (base-runtime, z-image, trellis, z-image-trainer, brain) with install scripts, status checks, service definitions, and UI metadata. It is the source of truth for the Manager's internal addon system but is static per release.

---

## 2. Why Registry Versioning Should Be Implemented

### 2.1 Problem: Coupled Release Cycle

**Today:** Adding a new module to the ecosystem requires:
1. Update `nymphs.json` on GitHub
2. Update bundled `addons.json` in the Manager codebase
3. Build and release a new Manager version
4. Users update Manager to see the new module

**With registry:** Steps 2-4 are eliminated. Manager fetches the registry at runtime and discovers new modules immediately.

### 2.2 Problem: No Lightweight Change Detection

**Today:** Manager must SSH into WSL and run git commands on each cloned repo to check for updates. This is:
- Slow (multiple git fetch operations)
- Fragile (depends on WSL being running, repos being properly initialized)
- Blind to modules not yet installed (can't check for updates on repos that don't exist locally yet)

**With registry versioning:** Manager makes a single HTTP request, compares one integer (`registry_version`), and knows instantly if anything changed. No WSL dependency for the discovery phase.

### 2.3 Problem: No Module Discovery for New Modules

**Today:** A user can only know about modules that are hardcoded into their Manager version. If a new module (e.g., "LoRA Studio") is published, the user has no way to discover it without updating Manager.

**With registry:** Manager can present a "Browse Modules" or "Available Addons" screen populated from the registry, showing modules not yet installed with their descriptions, channels, and trust status.

### 2.4 Problem: No Central Authority

**Today:** Each module's GitHub repo is an independent source of truth. There's no single place to answer "what modules exist?" or "which are trusted?" or "which are on stable vs. test channels?"

**With registry:** The registry is the single source of truth for module metadata. The `trusted` and `channel` fields enable filtering and safety gates.

---

## 3. Recommended Registry Schema

### 3.1 Top-Level Fields

| Field | Type | Purpose |
|---|---|---|
| `registry_version` | integer | Monotonic change counter. Increment on every change to the registry. |
| `updated` | string (date) | Human-readable last-modified date. |
| `modules` | array | List of module entries. |

### 3.2 Module Entry Fields

| Field | Type | Required | Purpose |
|---|---|---|---|
| `id` | string | Yes | Unique identifier (lowercase, hyphenated). Used as the key for caching. |
| `name` | string | Yes | Display name. |
| `channel` | string | Yes | `"stable"`, `"test"`, `"beta"` — filters visibility. |
| `trusted` | boolean | Yes | Whether the module is vetted by Nymphs Nerds. Gate for auto-install. |
| `manifest_url` | string | Yes | URL to the module's `nymph.json` manifest. |
| `manifest_version` | string | No | Cached version from the manifest. Avoids fetching manifest just to display version. |
| `manifest_hash` | string | No | SHA-256 of the manifest. Detects manifest changes without re-downloading. |

### 3.3 Example with Enhanced Fields

```json
{
  "registry_version": 2,
  "updated": "2026-05-07",
  "modules": [
    {
      "id": "worbi",
      "name": "WORBI",
      "channel": "stable",
      "trusted": true,
      "manifest_url": "https://raw.githubusercontent.com/nymphnerds/worbi/main/nymph.json",
      "manifest_version": "1.2.0",
      "manifest_hash": "abc123..."
    }
  ]
}
```

---

## 4. Manager-Side Implementation Requirements

### 4.1 Registry Cache

**File:** `~/.local/share/NymphsCore/registry-cache.json`

```json
{
  "cached_registry_version": 1,
  "cached_at": "2026-05-07T20:30:00Z",
  "modules": {
    "worbi": {
      "manifest_version": "1.2.0",
      "installed": true,
      "last_checked": "2026-05-07T20:30:00Z"
    }
  }
}
```

### 4.2 Update Detection Flow

```
Manager Startup / Periodic Refresh:
┌─────────────────────────────────────────────┐
│ 1. Fetch nymphs-registry/nymphs.json        │
│    (HTTP GET, conditional with If-None-Match)│
├─────────────────────────────────────────────┤
│ 2. Compare remote.registry_version          │
│    vs cached.cached_registry_version        │
├─────────────────────────────────────────────┤
│ 3a. Same → skip (304 Not Modified)          │
│ 3b. Different → parse, diff, update cache   │
├─────────────────────────────────────────────┤
│ 4. For each module with changed             │
│    manifest_hash → fetch manifest, compare  │
│    manifest_version                         │
├─────────────────────────────────────────────┤
│ 5. Present update notifications:            │
│    - "X modules have updates"               │
│    - "Y new modules available"              │
│    - Badge on module cards                  │
└─────────────────────────────────────────────┘
```

### 4.3 New Manager Components

| Component | Type | Responsibility |
|---|---|---|
| `RegistryService` | Service | Fetch, parse, cache, and diff the registry. |
| `RegistryViewModel` | ViewModel | Expose module list, update states, install commands to the UI. |
| Module browse step | UI Step | New installer step or Runtime Tools tab for browsing/discovering modules. |
| `RefreshRegistryCommand` | Command | Manual refresh button, same as `RefreshRuntimeStatusCommand` pattern. |

### 4.4 Integration Points

- **Runtime Tools page:** Add a "Modules" section showing installed modules with update badges, plus available-not-installed modules.
- **System Check page:** After detecting existing install, also check registry for new modules and show "X new modules available" in the summary.
- **Settings:** Option to filter by channel (hide test-channel modules unless opted in).

---

## 5. Implementation Priorities

### Phase 1: Registry Infrastructure (Foundation)
- [ ] Add `manifest_version` and `manifest_hash` to registry entries
- [ ] Implement `RegistryService` in Manager (fetch, parse, cache)
- [ ] Wire registry fetch on Manager startup (non-blocking, background)
- [ ] Store registry cache on disk

### Phase 2: Update Notification
- [ ] Compare cached vs. remote registry
- [ ] Show update badge/indicator in Runtime Tools
- [ ] Per-module "update available" state

### Phase 3: Module Discovery UI
- [ ] Browse/Discover modules screen
- [ ] Install from registry flow
- [ ] Channel filtering (stable/test/beta)
- [ ] Trusted module gate

### Phase 4: Deprecation Path
- [ ] Migrate `addons.json` metadata to registry manifests
- [ ] Remove bundled `addons.json` from Manager
- [ ] Registry becomes the single source of truth

---

## 6. Risk Assessment

| Risk | Likelihood | Mitigation |
|---|---|---|
| Registry GitHub raw URL goes down | Low | Cache last-known-good, retry with backoff, show stale data with warning |
| Manifest URL returns invalid JSON | Medium | Per-module error isolation, don't fail entire registry parse |
| User on unstable network | Medium | Conditional requests (If-None-Match), throttle refresh frequency |
| Untrusted module in registry | Low | `trusted` boolean gate, require trusted=true for auto-install |

---

## 7. Decision Required

**Should Manager consume the registry at runtime, or keep the current bundled `addons.json` approach?**

Recommendation: **Yes, consume the registry.** The bundled approach works for a closed ecosystem of 5 modules but does not scale to a marketplace. The registry versioning system is the correct architectural choice for a modular, extensible platform. The implementation cost is moderate (Phase 1 is ~1-2 days of work) and the payoff is decoupling module discovery from Manager releases.

---

## 8. Files Referenced

| File | Location | Role |
|---|---|---|
| `nymphs.json` | `nymphs-registry/` | Registry index (this proposal's focus) |
| `addons.json` | Manager source, bundled | Current static addon metadata |
| `MainWindowViewModel.cs` | Manager source | `RunManagedRepoUpdateCheckAsync()` — existing update flow |
| `InstallerWorkflowService.cs` | Manager source | Git-based repo check logic inside WSL |

---

*End of handoff document.*