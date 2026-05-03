# Firefox Support — Design Spec

**Status:** Draft, awaiting decisions
**Predecessor:** `docs/superpowers/research/2026-05-03-firefox-support-feasibility.md`
**Goal:** Ship Pounce as a single-source extension that builds for both Chrome (Web Store) and Firefox (AMO).

---

## Architecture decisions

### A1. Single source + build-time manifest swap

`manifest.json` stays Chrome-shaped (current state). Firefox-specific manifest fragments live in `manifest.firefox.json` and are merged at build time.

```
manifest.json              # Chrome MV3, untouched
manifest.firefox.json      # Firefox MV3 overrides
build.sh                   # accepts --target chrome|firefox (default chrome)
```

`build.sh` uses `python3 -c 'json.load... json.dump...'` to deep-merge the override on top of the base when `--target firefox`, then writes the merged result into the zip as `manifest.json`. No JS / no extra build dep.

**Rejected alternatives:**
- **Fork branch.** High maintenance, drifts. Killed.
- **Per-target full manifest.** Two files to keep parity. Killed.
- **Webpack/Rollup.** Adds a bundler to a no-bundler project. Killed.

### A2. Background script shape

Pounce uses `background.service_worker` for Chrome. Firefox 121+ supports `service_worker`, but ESR 115 (the recommended floor) only honors `background.scripts`. Path forward:

- For Firefox build, the merged manifest replaces `background` with `{ "scripts": ["background.js"], "type": "module": false }`.
- `background.js` is already written to be re-entrant (each `chrome.notifications.create` re-loads i18n) so works under both event-page and service-worker lifecycle.

### A3. `chrome.topSites` handling

Drop silently. `getTopSitesData()` already returns `[]` on throw. Firefox build adds a one-line note in the source comment; no other change.

**Why not history-derived fallback now:** YAGNI. Ship the smallest Firefox build first; if Firefox users miss it, add the fallback in a follow-up. The cost of adding it later is one function and one PR.

### A4. Distribution

**AMO submission is mandatory** for end-user installs on Firefox release channel — the browser refuses unsigned extensions. Self-hosted `.xpi` only runs on Developer Edition / Nightly / ESR-with-policy. So either:

- (a) Submit to AMO. First review usually < 24h; updates are usually instant after auto-review passes.
- (b) Don't ship for Firefox. Defeats the point.

→ **Default: AMO.** Build artifact is `pounce-X.Y.Z.xpi` (same zip format, different extension).

### A5. Minimum Firefox version

`browser_specific_settings.gecko.strict_min_version = "115.0"`. Reasoning:

- 115 is current ESR (used by enterprises and conservative users).
- All Pounce APIs we use (storage, tabs, bookmarks, history, scripting, commands, notifications) are stable on 115.
- Going lower (109 = MV3 GA) buys very few users and surfaces edge-case API gaps.

### A6. Keyboard shortcut conflicts

Firefox macOS hard-binds `Cmd+K` to the search bar. Pounce can't override. Resolution:

- Manifest-declared `suggested_key` is best-effort; Firefox honors it if not in conflict.
- README documents the rebind path: `about:addons` → ⚙️ → Manage Extension Shortcuts.
- We don't change the suggested key; users can rebind freely.

### A7. Bridge tab on `about:*`

The bridge tab pattern relies on opening an extension page (`bridge.html`) — not on injecting into `about:*` directly. Should work the same way as `chrome://*` does on Chrome. **Verified during smoke test, not by spec.** If it breaks, scope reduces to "Cmd+K works on regular pages, fails on `about:*`" — acceptable degradation, document in README.

---

## File map

| Path | Action | Why |
|---|---|---|
| `manifest.firefox.json` | Create | Gecko-specific overrides |
| `build.sh` | Modify | Add `--target` flag, manifest merge, `.xpi` extension for firefox |
| `README.md` | Modify | Add Firefox install section + shortcut rebind note |
| `README.zh-CN.md` | Modify | Mirror in Chinese |
| `docs/superpowers/plans/2026-05-03-firefox-support.md` | Create | Step-by-step exec plan |
| `tests/firefox-smoke.md` | Create | Manual smoke checklist |

No JS / HTML / CSS file changes. No new dependencies.

---

## `manifest.firefox.json` shape (preview)

```json
{
  "browser_specific_settings": {
    "gecko": {
      "id": "pounce@tuyv.dev",
      "strict_min_version": "115.0"
    }
  },
  "background": {
    "scripts": ["background.js"]
  }
}
```

`background` overrides the entire field (drops `service_worker`). All other fields stay from the base manifest.

---

## Open decisions — please confirm

1. **AMO submission:** OK to submit? (Required for any real Firefox install.)
2. **Drop `topSites` silently for Firefox:** OK? Alternative is to derive frequent-sites from history (one extra function, ~half day extra work).
3. **Min Firefox version 115 ESR:** OK? Alternative is 109 (MV3 GA) for wider reach but more edge cases.
4. **Gecko ID `pounce@tuyv.dev`:** OK as the permanent extension ID? Once submitted to AMO this can't change without forfeiting the listing.

---

## Out of scope (this round)

- Localization of AMO listing page (separate work in AMO dashboard).
- Cross-browser e2e CI (Pounce has no CI today; manual smoke is the bar).
- Vivaldi / Brave / Edge specifics (those are Chromium; current Chrome build works).
- Mobile Firefox (Pounce uses desktop-only APIs like `chrome.commands`).

---

## Next step

After decisions confirmed: write `docs/superpowers/plans/2026-05-03-firefox-support.md` and execute via `subagent-driven-development`.
