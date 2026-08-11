# Compatibility & versions

Honest notes on what we run, what worked, and what didn't. These are observations from one install
(a Luba 3), not authoritative guidance. The maintainers' release notes always win.

## Our baseline

| Component | Version | Notes |
|---|---|---|
| Mower | Mammotion Luba 3 | The only model we've tested. Reports welcome for Luba 2 / Yuka. |
| [`mikey0000/Mammotion-HA`](https://github.com/mikey0000/Mammotion-HA) | `0.6.4-beta9` | Installed via HACS. |
| [`pymammotion`](https://github.com/mikey0000/pymammotion) | `0.8.11` | Bundled by the integration. |
| Home Assistant | 2026.x | Templating + packages + Assist. |
| Transport | ESPHome Bluetooth proxies (+ WiFi/cloud fallback) | Good proxy coverage matters — see below. |

## The zone-switch churn and beta pinning

The entity-id re-enumeration described in [the-604-fix.md](the-604-fix.md) is the behavior we most
had to design around. When we audited whether a newer beta fixed it:

- Installed `0.6.4-beta9` (pymammotion 0.8.11); latest at the time was `0.6.4-beta11`.
- The `beta9 → beta11` delta was small (a pymammotion bump `0.8.11 → 0.8.12`, a "remove battery level"
  change, and version chores) and **did not touch the area/zone entity id stability**.
- [Issue #604](https://github.com/mikey0000/Mammotion-HA/issues/604) is the matching report; it's
  marked closed, but the orphan/re-enumeration churn still occurred for us.

**Our conclusion, for our situation:** stay on beta9 while the yard map is actively changing (each
map edit reshuffles entities anyway, and upgrading mid-construction risks another reshuffle), and
revisit newer betas once the map settles. The resolve-by-substring proxies make us largely indifferent
to the churn regardless of version, which is the point — **you shouldn't need a specific version to
have reliable zone automations.**

Please don't read this as "don't upgrade." It's "here's why *we* paused upgrades during active map
changes." Newer releases may well improve things; check the maintainers' notes.

## A word on firmware and app updates

We want to mention this plainly, without telling anyone what to do: **most of the breakage documented
in this repo was triggered by a mower firmware update or a change made through the Mammotion phone
app.** A few examples from our own experience:

- A **firmware/telemetry change** on our Luba 3 started making the battery percentage and docked state
  flap (reading 0 ↔ 80 every few seconds). That single change is why the battery-only "charged" signal
  stopped latching and why we moved to a [plug-based charge signal](../packages/mammotion_charge_plug.yaml).
- **Editing or recreating an area/map in the app** re-enumerates the zone switch entity-ids every time,
  which is the churn the [#604 fix](the-604-fix.md) exists to survive.
- App and firmware versions can shift what the integration sees, sometimes independently of the
  integration's own version.

We're **not** suggesting you should or shouldn't update — that's entirely your decision, and updates
often bring real improvements. We're only noting that, in our experience, these events have been the
usual *trigger* for things suddenly behaving differently. So if your setup was stable and then broke,
a recent firmware or app update is a reasonable first thing to consider. None of this is the
integration maintainers' doing — they're chasing a moving target on the other side of a closed,
changing protocol, which is exactly why a resilient HA-side layer helps.

## Things that worked

- **Config-entry reload as recovery.** 3/3 clean recoveries in ~1 minute each. This is the backbone
  of `script.mower_recover_integration`.
- **Resolve-by-available-substring** for zone switches — survives every remap we've seen.
- **Debounced derived signals** (`mower_charging`, `mower_charged`, `mammotion_integration_healthy`)
  as the basis for all flow control, instead of raw mower state edges.
- **Riding WiFi-resilient entities for feedback**, not BLE-only ones, so progress doesn't vanish
  during a BLE range drop (§10.22 in the lessons log).

## Things that did *not* work

- **The integration's "Sync Maps" button as a recovery step.** 0/5 in our testing — it timed out
  every time. We removed it from the recovery path entirely. (This says nothing bad about the
  feature's intended purpose; it just wasn't a reliable *recovery* lever for us.)
- **Hardcoding zone `entity_id`s.** Breaks silently on the next map change. Don't.
- **Triggering flow on raw `→ docked` edges.** Fires mid-dock-cycle and on `unavailable` glitches;
  false-advanced our multi-zone flow 6 seconds into the next mow.
- **Short recharge timeouts.** A full-yard deep discharge → 80% ready can take ~2.5 h; a 1.5 h
  timeout silently ended the flow. If you must time out a recharge wait, size it for worst-case.

## Transport / BLE notes

Several "integration dropped" incidents turned out to be **BLE proxy range loss**, not integration
faults — the mower drove out of proxy range mid-mow, and a reload once it returned to the dock
restored everything. If you see recoveries clustering around a particular part of the yard, suspect
proxy coverage before the integration. Good ESPHome Bluetooth proxy placement is the quiet
prerequisite for all of this.

## Model coverage

Everything here is Luba-3-tested. The patterns (substring resolution, reload-first recovery, derived
signals) should translate to any Mammotion model the integration supports, but the exact entity slugs
(`luba_3`, `activity_mode` values like `MODE_WORKING`, etc.) will differ. If you adapt this to another
model, a PR updating this table would help the next person.
