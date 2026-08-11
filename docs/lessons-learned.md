# Lessons learned

A distilled log of things we actually saw go wrong running a Mammotion mower unattended, and what
fixed them. It's abridged from a much longer private failure log (~40 incidents) down to the ones with
a **transferable** takeaway. Read this before designing your own flows — most of these are cheaper to
learn here than in your own yard.

None of these are complaints about the integration. They're the normal reality of automating a closed
mower protocol, and every fix lives on the Home Assistant side.

---

### 1. State-edge triggers on a flaky integration must filter glitch sources
**Saw:** a multi-zone flow ended prematurely and announced "complete" while the next zone was actually
starting. **Cause:** the integration glitched `docked → unavailable → docked` in a 38-second window,
and an advance automation triggered on `to: docked` regardless of the previous state. **Fix:** add
`not_from: [unavailable, unknown, none]` to every `to: docked` trigger. Better still, don't trigger on
the raw mower state at all — trigger on the **debounced ON-edge of a derived signal**.

### 2. Never hardcode a zone/area switch entity_id
**Saw:** "mow the front yard" spoke the confirmation but the mower never moved. **Cause:** the
integration re-prefixed/re-suffixed the zone switch entity_ids on a map sync; every hardcoded
reference 404'd silently. **Fix:** resolve by stable substring + available-filter. This is the whole
of [the-604-fix.md](the-604-fix.md) — the most important lesson in the repo.

### 3. A debounced ON-edge is the only trustworthy "task complete" latch
**Saw:** the "backyard done" advance fired **6 seconds into** the backyard mow — as the mower left the
dock. **Cause:** it triggered on the single raw edge `lawn_mower → docked`, which is indistinguishable
from a glitch or an in-progress dock cycle. **Fix:** gate on `binary_sensor.mower_charging` (a
docked-state latch with `delay_on/off: 10s`) turning ON — it settles ~10s after the mower is genuinely
home, never during transit.

### 4. Slow operations need their *own* progress signal
**Saw:** a recovery took ~8 minutes with zero user feedback; it looked hung. **Cause:** announcements
were buried inside the mow script, and the recovery branch had no narration of its own. **Fix:** give
slow steps (recovery, recharge) a dedicated status entity (`sensor.mammotion_integration_status`) and
let announcements watch *that*, independent of whatever script triggered the recovery.

### 5. Size recharge timeouts for the worst case, or remove them
**Saw:** "mow all lawns" mowed the front, then silently ended without the backyard. **Cause:** the
inter-zone wait for "charged" was capped at 1.5 h, but a deep discharge → 80%-ready cycle takes ~2.5 h;
the wait timed out and the flow fell through to cleanup. **Fix:** bump the backstop to 4 h — or, better,
move the wait *out of the script* into an automation that triggers on the "charged" off→on edge, which
needs no timeout at all.

### 6. Recovery reloads destroy in-flight derived signals — re-check after recovering
**Saw:** a mow aborted with "not ready" immediately after a *successful* recovery, mower docked at 80%
the whole time. **Cause:** the reload inside recovery reset the debounced `mower_charged` sensor's
timer, so it read `off` right after recovery even though the mower was ready. **Fix:** after calling
recovery, wait for the derived signals to re-settle before gating on them; don't assume they survived
the reload.

### 7. "Sync Maps" was never a reliable recovery lever (for us)
**Saw:** 5 recoveries attempted with Sync Maps first — 0 recovered, all timed out. A config-entry
reload recovered 3/3. **Fix:** recovery is **reload-first, Sync-Maps-never**, with a hard cap of two
server touches so we never risk a cloud lockout. See [self-recovery.md](self-recovery.md). (Sync Maps
may serve its intended purpose fine; it just isn't a recovery tool.)

### 8. `mowing` state ≠ actually mowing — wait for proof of motion
**Saw:** false "mowing now" announcements while the mower sat on its dock. **Cause:** the integration
flips to `mowing` during at-dock pre-roll (RTK fix, blade spin-up) before it moves. **Fix:** only
treat it as mowing once `sensor.luba_3_mowing_speed > 0` (held ~2s) and/or
`activity_mode == MODE_WORKING`. Model this as a distinct `starting` state.

### 9. Veto on rain *accumulation*, not instantaneous rate
**Saw:** a trace of drizzle (rate briefly > 0) skipped a whole mowing day. **Cause:** the wet-grass
veto keyed off instantaneous rain *rate*. **Fix:** veto on **accumulated** rainfall over a window
(we use ~0.03–0.06 inch), which reflects whether the grass is actually wet. See
[`packages/mammotion_rain_gate.yaml`](../packages/mammotion_rain_gate.yaml).

### 10. Ride WiFi-resilient entities for feedback, not BLE-only ones
**Saw:** mow progress vanished mid-mow whenever the mower drove out of BLE proxy range. **Cause:**
progress feedback was derived from BLE-updated entities. **Fix:** base user-facing status on entities
the integration keeps fresh over WiFi/cloud, so a BLE range dip doesn't blind the UI. Also: many
"integration dropped" events are really **proxy range loss** — check coverage before blaming the
integration.

### 11. Restart-proof anything long-running with persistent helpers + a start trigger
**Saw:** a mid-mow HA restart produced a wildly wrong completion duration, and per-yard durations
sometimes stopped recording. **Cause:** scripts killed by the restart never reached their
stop/duration step, and template sensors went `unknown` on reload. **Fix:** keep orchestration state
in `input_*` helpers (not script-local variables), add a `homeassistant: start` trigger to status
templates so they never go `unknown`, and have a "recovery on HA start" automation resume the flow by
phase. Never run `script.reload` or restart HA mid-mow — it cancels the running script and kills its
completion logging.

### 12. Don't announce during churn, and gate background voice on quiet hours
**Saw:** false "mowing / canceled / recovery failed" chatter during integration flapping, sometimes at
night. **Cause:** announcements fired on raw, un-debounced states, with no quiet gate. **Fix:** drive
announcements off the debounced status sensor, suppress them during integration churn, and route
*background* announcements through a Do-Not-Disturb gate that still posts the notification but skips
the voice at night. (The immediate reply to a voice command you just issued always speaks — you
initiated it.)

---

**The meta-lesson:** build **debounced, derived status entities** as the single source of truth, and
make every script, automation, dashboard, and announcement a *consumer* of them. Almost every fix
above is a variation on "stop reacting to the integration's raw, glitchy state; react to a signal you
smoothed and verified yourself."
