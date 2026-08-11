# Self-recovery: health, status, and the reload-first recovery script

The second reason this system runs unattended: when the integration drops its controls, it **notices
and recovers itself**, then reports honestly whether it worked. This doc explains the sensor stack and
the recovery routine, both in [`packages/mammotion_core.yaml`](../packages/mammotion_core.yaml).

## The failure this addresses

The Mammotion integration can enter a state where **the core sensors stay up but the zone controls go
`unavailable`** — "sensors up, controls gone." A naive `is_state(mower, 'unavailable')` check misses
it entirely, so an automation sails past its gate and then stalls trying to select a zone that isn't
there. We needed a boolean that means *"core **and** controls are up"*, not just "the mower entity
exists."

## The signal stack

Build these bottom-up; each layer consumes the one below.

### `binary_sensor.mammotion_integration_healthy` — the gate
ON only when the mower entity is available **and** all three zone display proxies
([the #604 fix](the-604-fix.md)) report available. Debounced (`delay_on ~15s`, `delay_off ~10s`) so a
momentary flap doesn't trip it. **This is the boolean every script gates on.** It's what catches the
"sensors up, controls gone" case a plain unavailable-check can't.

### `sensor.mammotion_integration_status` — the human-facing view
A multi-state, debounced sensor evaluated **healthy-first** (so a stale recovery phase can't mask a
recovered integration):

`Ready` › `Enabling` / `Reloading` / `Reconnecting` (current recovery step) › `Recovery Failed`
(gave up — needs a human) › `Offline` (core down, not recovering) › `Not Ready` (core up, controls
down, no recovery running).

Attributes carry `attempt`, `since`, and a human `detail` line ("Reloading — waiting for the
integration · attempt 2"). This is what a dashboard hero card reads.

### `sensor.mower_status` — the one entity your UI and voice should read
A trigger-based template with a `homeassistant: start` trigger (so it survives reloads and never goes
`unknown`) and a short debounce. Priority-ordered:

`offline` › `reconnecting` › `error` › `stuck` › `paused` › `returning` › `starting` (state is
mowing but not yet moving — at-dock pre-roll) › `mowing` › `ready` (docked + charged) › `charging` ›
`docked`.

The `starting` vs `mowing` distinction matters: the integration flips the mower to `mowing` during
**at-dock pre-roll** (RTK fix, blade spin-up) *before* it physically leaves. We only call it `mowing`
once there's proof of motion (`sensor.luba_3_mowing_speed > 0` and/or `activity_mode == MODE_WORKING`).
Announcing on the raw `mowing` state gives you false "mowing now" messages while the mower sits still.

### The "task complete" latch — `binary_sensor.mower_charging` / `mower_charged`
Two debounced derived signals that replace racy raw state edges:
- **`mower_charging`** = `is_state(mower, 'docked')` with `delay_on: 10s`, `delay_off: 10s`. It is a
  *home/docked* latch (named for the robovac convention), ON the whole time the mower is home. Its
  **ON-edge, ~10s after the mower settles**, is the authoritative "zone/task complete" signal — not
  the raw `→ docked` edge, which also fires mid-dock-cycle and on glitches.
- **`mower_charged`** = `docked AND battery ≥ 80` held `delay_on: 5 min`. ON = "charged and ready for
  the next task." Multi-zone flows advance off *this*, not off a raw battery reading.

> Why derived signals instead of raw edges? Because `docked → unavailable → docked` flickers on
> reload, and the raw `→ docked` edge fires 6 seconds into the *next* mow. Debounced ON-edges of
> derived booleans are the only stable "it really finished" signal. See
> [lessons-learned.md](lessons-learned.md) §10.1, §10.3.

## The recovery script: `script.mower_recover_integration`

A standalone, reusable routine (`mode: single`) callable from any script. It's **reload-first**, and
its most important property is a **hard cap of two "server touches"** — we never hammer the mower's
cloud endpoint, which risks a lockout.

The design came straight from data: over five real recoveries, the integration's own **"Sync Maps"
button recovered it 0 out of 5 times**, while a **config-entry reload recovered it 3 out of 3** (the
other two were a different, intentionally-disabled case). So Sync Maps is **not in the recovery path
at all**. See [lessons-learned.md](lessons-learned.md) §10.9.

Flow:
1. **Already healthy?** → do nothing, reset the phase, exit.
2. Bump the attempt counter.
3. **Branch** on `input_boolean.mammotion_intentionally_disabled`:
   - **Path A (was intentionally disabled** — e.g. you disabled it to control the mower from the
     phone app): enable the config entry, then wait for `activity_mode` to return (3 min) and
     `mammotion_integration_healthy` to flip on (2 min); one reload as a fallback.
   - **Path B (was enabled but unhealthy** — the common "controls dropped out" case): reload the
     config entry; wait; **one** reload retry if still unhealthy.
4. **Result:** phase → `idle` on success, or `failed` if it gave up. The script **does not announce or
   clean up** — that's the caller's job, so the same recovery routine works for a mow script, a manual
   "recover the mower" button, or a periodic watchdog.

Throughout, it publishes its current step to `input_select.mammotion_recovery_phase`, which drives
`sensor.mammotion_integration_status` so the UI shows live progress.

### Two things you must configure
- **`!secret mammotion_config_entry_id`** — the reload needs your integration's config-entry id. Find
  it at **Settings → Devices & Services → Mammotion → ⋮ → (the URL contains the entry id)**, or via
  Developer Tools, and add it to `secrets.yaml`:
  ```yaml
  mammotion_config_entry_id: "0123456789abcdef0123456789abcdef"
  ```
- **The enable path (Path A) is optional.** `homeassistant.reload_config_entry` is a core service, but
  re-*enabling* a disabled entry historically needed the [Spook](https://github.com/frenck/spook)
  add-on's `homeassistant.enable_config_entry`. If you never disable the integration for phone
  control, you can delete Path A entirely and keep only the reload path.

## Timeouts: backstops, not control mechanisms

Home Assistant's `wait_template` is **event-driven** under the hood — it listens for state changes and
only re-evaluates then. So "wait_template with a timeout" really means "wait for the event, with a
safety backstop." Most of our waits fall into this category: the **event** is the control mechanism,
the **timeout** is just "we won't wait beyond reasonable." Set those generously.

The exception is inside the recovery script, where the timeout genuinely *is* the decision — a reload
emits no "done" event, so we infer recovery from `activity_mode` returning and `healthy` flipping on,
*or* the wait timing out. Those few timeouts are load-bearing; the rest are backstops. Worth keeping
the distinction in mind when you tune anything.
