# mammotion-ha-resilient

A set of Home Assistant packages, scripts, and notes that make a **Mammotion robotic mower**
reliable enough to run unattended — resolving the zone-switch churn that trips up automations,
recovering the integration by itself when it drops out, and (optionally) driving the whole thing
by voice.

Everything here is shared humbly, in the hope it saves someone else the weeks of trial and error
it took us. **None of it would exist without the [`mikey0000/Mammotion-HA`](https://github.com/mikey0000/Mammotion-HA)
integration and the [`pymammotion`](https://github.com/mikey0000/pymammotion) library** — this
repository is a thin layer of *userspace* configuration on top of their work. Thank you. 🙏

> **This is not a replacement for the Mammotion integration.** You install their integration first
> (via HACS). This repo is the automation glue we built *around* it. If anything here conflicts with
> the maintainers' guidance, follow theirs.

> **This solution is not perfect.** It doesn't fix the underlying problems — it works *around* them.
> What we can honestly say is that it's **reliable and runs well in daily use** despite all the rough
> edges described below. Expect to adapt it to your own mower, sensors, and yard rather than dropping
> it in untouched.

> **Free, for anyone, no strings — and use at your own risk.** This is given away under the
> [MIT license](LICENSE). We're not selling anything and there's nothing to sign up for. In exchange,
> understand that it comes with **no warranty**: it's imperfect, it controls a machine with spinning
> blades, and you are responsible for testing it and operating it safely. See the
> [disclaimer](#contributing--disclaimer) at the bottom.

---

## Why this exists

Talking to a closed, ever-changing mower protocol over BLE and cloud is genuinely hard, and the
community integration does an impressive job of it. But a few rough edges make it awkward to build
*unattended* automations on top:

1. **Zone/area switch entity-ids are not stable.** The integration re-enumerates the per-zone
   `switch.…_area_*` entity-ids on every map sync or area edit (appending a numeric suffix and
   orphaning the old entity to `unavailable`). Any automation that hardcodes a zone switch breaks
   silently the next time the map changes — the mower "accepts" the command and never moves. This is
   [issue #604](https://github.com/mikey0000/Mammotion-HA/issues/604) (closed, but the entity churn
   persists as of the versions we run).
2. **The integration occasionally drops its controls** — the core sensors stay up, but the zone
   switches go `unavailable`. A plain `unavailable` check doesn't catch it, so an automation sails
   past the gate and stalls.
3. **State edges are glitchy.** `docked → unavailable → docked` flickers on reload; the raw
   `→ docked` edge fires mid-transit. Naive triggers false-advance multi-zone flows.

None of these are criticisms of the integration — they're the reality of the underlying protocol.
This repo is a collection of patterns that work *around* them from the HA side, so you don't have to
rediscover them.

## What's in here (and what actually works)

The system is layered. **You can stop at any layer**, and everything past the base is a genuinely
optional extra you add only if you want it.

### The reliable base

| Layer | File | What it gives you | Depends on |
|---|---|---|---|
| **1. Core reliability** | [`packages/mammotion_core.yaml`](packages/mammotion_core.yaml) | Stable, remap-proof zone proxies (the #604 fix) · a self-healing recovery script · health + rich status sensors | Mammotion integration only |
| **2. Orchestration** | [`packages/mammotion_orchestration.yaml`](packages/mammotion_orchestration.yaml) | Per-yard mow scripts + a "mow the whole yard" flow that charges between zones and survives HA restarts | Layer 1 |

**Honestly, for most people the value is Layer 1.** If you only ever copy `mammotion_core.yaml`, the
zone-switch fix alone will make your existing automations stop breaking on every map change.

### Optional extras (add any, all, or none)

| Extra | File / folder | What it adds | Depends on |
|---|---|---|---|
| **Charge plug** | [`packages/mammotion_charge_plug.yaml`](packages/mammotion_charge_plug.yaml) | A rock-steady, restart-proof "charge complete" signal from a smart plug on the dock — the upgrade that made *our* setup dependable, because our mower's battery telemetry flaps. Optional, but recommended if yours flaps too. | Layer 1 + a power-metering smart plug |
| **Rain gating** | [`packages/mammotion_rain_gate.yaml`](packages/mammotion_rain_gate.yaml) | A generic "don't mow wet grass" veto (accumulation-based, wired to your own rain sensor) | Layer 1 + your rain sensor |
| **Voice** | [`voice/`](voice/) | Home Assistant Assist voice control + quiet-aware spoken announcements | Layers 1–2 |
| **Smart Mow (VPD)** | [`examples/`](examples/vpd-scheduling-example.md) | Our fully-automatic scheduler that decides *when* to mow from vapor-pressure deficit + per-yard cadence + rain veto. Shipped as a **featured reference + walkthrough** (verbatim config you adapt), not a turnkey install — it's tied to our specific climate sensors. | Your own climate/soil/rain sensors |

> **A note on the gate.** In our own yard one lawn sits behind a physical garden gate, so our flow
> checks a gate sensor before mowing that zone. We deliberately **left that out of the shipped
> scripts** to keep them clean and universal — but if a zone of yours is behind a gate or door, the
> one-block pattern to add it back is documented in
> [`docs/gate-precondition.md`](docs/gate-precondition.md).

### What this does *not* do
- It does not modify or patch the integration. It's pure HA configuration (templates, scripts,
  automations, helpers).
- It does not make a flaky BLE link reliable — it recovers *cleanly* when the link drops, but good
  BLE proxy coverage is still on you.
- The VPD scheduling and the specific rain sensors are **tied to our hardware**. They're included as
  documented examples to adapt, not turnkey installs.

## Versions we run (as of this writing)

These are the versions this was built and tested against — not a recommendation to pin, just honesty
about our baseline. Newer releases from the maintainers may already improve on things noted here.

| Component | Version |
|---|---|
| Mower | Mammotion Luba 3 |
| `mikey0000/Mammotion-HA` | `0.6.4-beta9` |
| `pymammotion` | `0.8.11` |
| Home Assistant | 2026.x |

A note on the zone-switch churn: in our testing the `beta9 → beta11` delta didn't change the
entity-id re-enumeration behavior, so we stay on beta9 while our yard map is actively changing and
revisit newer betas once the map settles. **This is our situation, not advice** — please follow the
maintainers' release notes. See [`docs/compatibility.md`](docs/compatibility.md) for details.

## Quick start (Layer 1)

1. Install the [Mammotion integration](https://github.com/mikey0000/Mammotion-HA) via HACS and get
   your mower online first. Confirm you have a `lawn_mower.*` entity and per-zone `switch.…_area_*`
   entities.
2. Make sure packages are enabled in `configuration.yaml`:
   ```yaml
   homeassistant:
     packages: !include_dir_named packages
   ```
3. Copy [`packages/mammotion_core.yaml`](packages/mammotion_core.yaml) into your `config/packages/`
   folder.
4. Edit the two things the file's header calls out:
   - the **zone-name substrings** (so the proxies match *your* area names), and
   - the **`mammotion_config_entry_id`** secret (so recovery can reload your integration).
5. If your mower's entity slug isn't `luba_3`, do a find-and-replace to match yours (the header
   explains where the slug comes from).
6. Restart Home Assistant. You should now have stable `binary_sensor.mow_area_*`,
   `sensor.mower_status`, and `binary_sensor.mammotion_integration_healthy` entities.

Then add Layers 2–4 as you like — each file's header lists its own steps.

## Documentation

- **[docs/the-604-fix.md](docs/the-604-fix.md)** — the zone-switch churn, and the resolve-by-substring
  pattern that survives it. The single most useful idea in this repo.
- **[docs/self-recovery.md](docs/self-recovery.md)** — the health/status sensor stack and the
  reload-first recovery script.
- **[docs/compatibility.md](docs/compatibility.md)** — versions, what worked / didn't, beta notes.
- **[docs/lessons-learned.md](docs/lessons-learned.md)** — a distilled log of what we actually saw go
  wrong, and what fixed it. Read this before designing your own flows.
- **[docs/gate-precondition.md](docs/gate-precondition.md)** — optional: gating a zone on a gate/door
  sensor, if one of your lawns is behind one.
- **[voice/README.md](voice/README.md)** — the optional voice layer.
- **[examples/vpd-scheduling-example.md](examples/vpd-scheduling-example.md)** — advanced scheduling
  by climate, as inspiration.

## A note to the Mammotion-HA maintainers

If you've found your way here: **thank you for the integration.** This repo exists only because your
work made a Mammotion mower controllable from Home Assistant at all. Everything here is a userspace
workaround, offered in case any of it is useful upstream — especially the zone-switch resolution
pattern in [`docs/the-604-fix.md`](docs/the-604-fix.md), which sidesteps the entity-id churn without
touching the integration. We'd be glad to see it made unnecessary by a stable-`unique_id` fix in the
integration itself. No obligation, and no criticism intended — just gratitude and a pattern that
might help.

## Acknowledgments

- **[mikey0000](https://github.com/mikey0000) and every contributor to `Mammotion-HA` and
  `pymammotion`** — the foundation this is built on. Thank you.
- **The Home Assistant community** — for the platform, the templating engine, and years of shared
  wisdom in the forums.
- **Claude and Anthropic** — this repository was designed, genericized, and documented with the help
  of Claude. It genuinely wouldn't exist in this form without that assistance.

## Contributing & disclaimer

Issues and PRs are welcome, especially reports of how this behaves with other Mammotion models
(Luba 2, Yuka, etc.) — we've only run it on a Luba 3.

This is given away freely and provided **as-is, with no warranty, under the [MIT license](LICENSE)** —
**use it entirely at your own risk.** It is imperfect: it works *around* the underlying problems rather
than fixing them. A robotic mower has spinning blades; you are responsible for safe operation,
boundaries, and no-go zones. Test every automation with the blades disabled first. Nothing here
overrides the safety guidance from Mammotion or the integration maintainers.

Related reading: much of what this works around traces back to mower firmware and phone-app changes —
see [docs/compatibility.md](docs/compatibility.md#a-word-on-firmware-and-app-updates) for a neutral note
on that (we don't tell you whether to update; we just note these have been the usual trigger for
breakage).
