Fork-local draft only. The issue and PR texts below are drafts for review and are NOT to be posted anywhere without approval. This file stays in the fork and must not be included in any upstream PR branch.

---

## Draft GitHub issue

**Title:** Expose host offline-render state to plugins (bounce/export/freeze)

**Body:**

### Problem

Many plugins can afford more expensive, higher-quality processing when the host is rendering offline — bounce, export, freeze, render-in-place — than during realtime playback: oversampling, longer reverb tails computed exactly, higher-precision filters, and so on. During realtime playback the same plugins need to stay cheap.

Hosts do expose this state, but through format-specific channels that dplug 16.5.6 currently ignores, so a `Client` implementation has no way to know:

| Format | Host signal | Status in dplug 16.5.6 |
|---|---|---|
| VST3 | `ProcessSetup.processMode` / `ProcessData.processMode == kOffline` | ignored |
| CLAP | `clap.render` extension (`set` with `CLAP_RENDER_OFFLINE`/`CLAP_RENDER_REALTIME`) | not implemented |
| VST2 | `audioMasterGetCurrentProcessLevel` returning `kVstProcessLevelOffline` (4) | not queried |
| AUv2 | `kAudioUnitProperty_OfflineRender` (property ID 37) | setter is a no-op |
| LV2 | `lv2:freeWheeling` designated control port | not declared in the plugin TTL |
| AAX | n/a — out of scope; dplug has no AAX client | — |

Note that for VST3 the mode can change without the host calling `setupProcessing` again, so it has to be re-read from `ProcessData` on every `process` call.

### Proposal

- `Client.isOfflineRendering() nothrow @nogc` — safe to call from the audio thread. Backed by an atomic; no lock, no allocation.
- `Client.setOfflineRendering(bool)` — for format clients only, called from the format glue.
- `TimeInfo.offline` — filled per block by each format client, mirroring how `hostIsPlaying` is filled.
- Conservative default: `false` whenever the host does not report an offline state. Plugins should never enable expensive processing on a guess.
- Changing the value does **not** force a `reset()`. Plugins that need to re-prepare internal state (buffer sizes, latency, precomputation) can watch the value and decide for themselves.

For precedent, this mirrors JUCE's `AudioProcessor::isNonRealtime()`.

### Known host limitations

Hosts do not notify consistently, which is why the state is re-read per block and defaults to `false`:

- Some VST2 hosts never set the process level at all (`audioMasterGetCurrentProcessLevel` is optional); the level is then unknown and treated as realtime.
- Some hosts notify the switch late — Cubase, for example, may only reflect the new mode at the next block boundary — so a value cached at initialization goes stale.
- The value can change mid-session (a user can start a bounce while the plugin keeps running), so it must be polled per block rather than assumed fixed.
- No thread-priority heuristics: unlike JUCE, which falls back to checking thread priority when the host doesn't report, dplug only reports what the host actually says.
- REAPER only reports offline rendering to VST plug-ins if the "Inform VST plug-ins of offline rendering" preference is enabled.

### Implementation status

All six format clients are wired: VST3, CLAP, VST2, AU and LV2 read the host state as described above; FLP documents that the FL SDK exposes no offline state (FL plugins always run realtime from the plugin's point of view).

VST3 and CLAP are covered by unit tests on Windows/LDC. VST2, AU and LV2 are compile-reviewed only at this point, pending access to the respective SDKs/platforms.

### Testing notes

- VST3/VST2: REAPER Render (with the preference above enabled), Bitwig Export.
- CLAP: `clap-validator`, Bitwig.
- LV2: `jalv --freewheel`.

---

## Draft PR description

**Title:** Expose host offline-render state to plugins (bounce/export/freeze)

**Body:**

### Summary

Adds `Client.isOfflineRendering()` and `TimeInfo.offline`, so plugins can switch to more expensive, higher-quality processing when the host renders offline (bounce/export/freeze) and stay cheap during realtime playback. Each format client reads the host's offline state and publishes it to the `Client` per block. Resolves #(issue).

### Per-format wiring

- **VST3:** `setupProcessing` reads `setup.processMode == kOffline`; `process` re-reads `data.processMode` per block, since hosts may switch modes without calling `setupProcessing` again.
- **CLAP:** implements the `clap.render` extension (`set` maps `CLAP_RENDER_OFFLINE`/`CLAP_RENDER_REALTIME`; `has_hard_realtime_requirement` returns `false`); `TimeInfo.offline` filled per block.
- **VST2:** `audioMasterGetCurrentProcessLevel` polled per block; `kVstProcessLevelOffline` (4) means offline, any unknown level (0) is treated as realtime.
- **AU:** `kAudioUnitProperty_OfflineRender` (37) — the property setter was previously a no-op, it now stores the value and the getter reads it back; `TimeInfo.offline` filled.
- **LV2:** declares an `lv2:freeWheeling` designated input control port (connectionOptional, notOnGUI, appended last in the TTL); `run` reads it per block.
- **FLP:** the FL SDK exposes no offline-render state; documented as always realtime.

### API contract

- `isOfflineRendering()` is `nothrow @nogc` and safe from the audio thread; the backing flag is a plain `shared(bool)` atomic.
- `setOfflineRendering(bool)` is for plugin format clients only, called from the format glue.
- No allocation, no lock.
- Default is `false` when the host doesn't report anything — plugins never enable expensive processing on a guess.
- Changing the value does not force a `reset()`; plugins decide whether to re-prepare.

### Tested

- `dub test :client` — 4 modules pass.
- `DFLAGS="-d-version=VST3 -d-version=CLAP -unittest" dub test :formats` — 26 modules pass. (Note for Linux shells; on Windows PowerShell the equivalent is `$env:DFLAGS = "-d-version=VST3 -d-version=CLAP -unittest"; dub test :formats`.)
- (Windows, LDC.)

### Compile-review only

VST2 (no SDK available here), AU (macOS) and LV2 (Linux) are reviewed for correctness but not executed in this environment; the wiring follows the respective specs and mirrors the tested paths.

### Manual test notes

- VST3/VST2: REAPER Render (enable "Inform VST plug-ins of offline rendering" in REAPER preferences), Bitwig Export.
- CLAP: `clap-validator`, Bitwig.
- LV2: `jalv --freewheel`.
