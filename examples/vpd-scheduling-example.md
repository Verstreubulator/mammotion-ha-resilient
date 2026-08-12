# VPD-driven mow scheduling — a worked example (not a drop-in)

This is a walk-through of how we let **vapor-pressure deficit (VPD)** decide *when*
to mow each day, per lawn, on top of a cadence and a rain veto.

**Its purpose in one line:** estimate **when the grass is dry enough to mow** as
conditions change — through the hours of a day, across variable weather, and from
season to season — instead of trusting a fixed clock time that's only ever right for
a few weeks of the year. The same VPD threshold that fires mid-morning in dry August
heat naturally waits until afternoon on a damp October day, with no rescheduling on
your part: the physics does the adjusting.

It is shared as **inspiration, not a package to install** — every entity name here is
ours, and the logic is tuned to our two lawns, our sensors, and our Luba 3. Read it
for the ideas, then adapt to whatever you have.

The full annotated source is next to this file:
[`mow_vpd_driven.reference.yaml`](./mow_vpd_driven.reference.yaml). That file is
labelled REFERENCE ONLY for the same reason.

> This all runs on top of the community integration
> [`mikey0000/Mammotion-HA`](https://github.com/mikey0000/Mammotion-HA) and the
> `pymammotion` library. Thank you to the maintainers and contributors — this is
> just userspace scheduling glue around what they provide.

---

## The idea in one paragraph

Grass cuts cleanest, and clippings mulch best, when the blades are **dry**. Rather
than pick a fixed clock time and hope, we watch VPD — a physics-based "how thirsty
is the air" number (kPa) that tracks drying far better than temperature or humidity
alone. Each lawn gets its own **VPD threshold** (the front is sunnier and dries
earlier than our partly-shaded back), a **cadence** (don't mow more often than every
N days), and a **finish-by deadline** (be docked before the quiet-hours / heat).
A once-a-minute dispatcher starts *one* lawn when it's **due AND VPD-ready AND dry
AND still feasible before the deadline**. "Mow both today" and "do half now, half
tomorrow" aren't coded — they *emerge* from repeatedly asking that question.

Why VPD instead of "has it rained": rain tells you water *fell*; VPD tells you
whether it has since *dried*. You want both — rain as a hard veto, VPD as the
go-signal. The rain veto is shipped separately and generically as
[`packages/mammotion_rain_gate.yaml`](../packages/mammotion_rain_gate.yaml)
(`binary_sensor.mow_rain_veto`); this example assumes something like it exists.

---

## First: where does a VPD sensor come from?

VPD isn't something your hardware reports — Home Assistant has no built-in VPD
sensor. You **calculate** it from two readings you almost certainly already have: an
air **temperature** and a **relative humidity**. The formula (air VPD, Tetens form):

```
SVP (saturation vapor pressure, kPa) = 0.61078 * e^( (17.27 * T) / (T + 237.3) )   # T in °C
VPD (kPa)                            = SVP * (1 - RH/100)
```

Higher VPD = drier, thirstier air = grass and dew dry faster. A copy-paste template
sensor that produces `sensor.outdoor_vpd` from your own temperature + humidity
entities is right next to this file:
[`vpd_sensor.yaml`](./vpd_sensor.yaml) — edit the two sensor names, handle °F vs °C,
reload. Everything below assumes you've created a VPD sensor that way (ours is
`sensor.outdoor_vpd_combined`, an average of a few spots; a single one is fine to
start).

## What it assumes you have

You would swap each of these for your own equivalents:

| Role | Our entity | Notes |
|------|-----------|-------|
| VPD (outdoor), kPa | `sensor.outdoor_vpd_combined` | The go-signal. Calculate it from temp + humidity — see [`vpd_sensor.yaml`](./vpd_sensor.yaml). |
| Rain veto | `binary_sensor.mow_lawn_wet`, `binary_sensor.mow_rain_significant` | See `mammotion_rain_gate.yaml` for a generic `binary_sensor.mow_rain_veto`. |
| The mower | `lawn_mower.luba_3`, `sensor.luba_3_*`, `number.luba_3_working_speed` | `luba_3` is our mower's slug — rename to yours. |
| Derived status | `sensor.mower_status`, `binary_sensor.lawn_mower_active`, `binary_sensor.mower_charged`, `sensor.mammotion_integration_status` | From the core packages in this repo. |
| Per-lawn mow scripts | `script.mow_the_front_yard`, `script.mow_the_backyard` | Your autonomous "go mow lawn X" routines. |
| Weather forecast | `weather.openweathermap` | Only for the optional "when will VPD cross" ETA. |
| Optional gate | `binary_sensor.gate` | A physical gate in front of one lawn — delete if you have none. |

Soil-moisture sensors are also referenced in our wider setup, but note: **root-zone
moisture is not the same as wet blades.** We deliberately gate mowing on VPD + rain,
not soil moisture. Soil moisture belongs to irrigation, not mowing.

---

## The building blocks (annotated snippets)

These are representative pieces from `mow_vpd_driven.reference.yaml`, not the whole
1000-line file. Line references point into that reference.

### 1. Per-lawn VPD threshold + a *restart-proof* "ready" gate

Each lawn is "ready" once VPD has been at/above its threshold for a sustained 20
minutes. The subtle part: we do **not** use a template `delay_on` for the 20-minute
sustain, because a `delay_on` timer **resets on every Home Assistant restart** — a
lunchtime restart once made us miss a 10 AM mow. Instead we stamp a **persisted**
timestamp when VPD first crosses above, and the sensor just compares against it:

```yaml
- name: "Mow Front VPD Ready"
  state: >
    {% set vpd = states('sensor.outdoor_vpd_combined') | float(-1) %}
    {% set thr = states('input_number.mow_vpd_threshold_front') | float(1.34) %}
    {% set since = as_timestamp(states('input_datetime.mow_front_vpd_above_since'), 0) %}
    {{ vpd >= thr and since >= 1577836800 and (as_timestamp(now()) - since) >= 1200 }}
```

- `float(-1)` fail-safe: if VPD is unavailable, ready reads **OFF** (never a stale ON).
- `since >= 1577836800` (2020-01-01) rejects an unset helper, so an un-stamped lawn
  never reads ready.
- `1200` seconds = the 20-minute sustain, measured from a timestamp that **survives
  restarts**.

A tiny companion automation stamps `mow_front_vpd_above_since` on a genuine
below→above crossing, and on HA start seeds it *only* if VPD is already above and the
stamp is unset — never overwriting a valid stamp. That's what makes the 20-minute
clock restart-proof.

### 2. Per-lawn cadence — "is this lawn due?"

```yaml
- name: "Mow Front Due"
  state: >
    {% set cad = 1 if is_state('input_boolean.mow_smart_every_day','on')
                  else states('input_number.mow_cadence_days') | float(2) %}
    {{ states('sensor.mow_front_days_since') | float(999) >= cad }}
```

`mow_front_days_since` is just "days since this lawn's stop-time was last stamped."
Any completed mow — scheduled, app-started, or manual — stamps it, so the cadence
clock stays honest no matter how the mow was launched.

### 3. Self-learning duration (so "finish by" is realistic)

We don't hardcode how long a lawn takes. Each completed mow nudges a stored estimate
via an exponential moving average, normalised to a 1 ft/s blade speed so it stays
valid when you change the mower's speed:

```yaml
value: >-
  {% set cur = states('input_number.mow_front_min_at_1fps') | float(91) %}
  {% set ws  = [states('number.luba_3_working_speed') | float(1.0), 0.3] | max %}
  {% set sample = (trigger.to_state.state | float(0)) * ws %}
  {{ (0.7 * cur + 0.3 * sample) | round(1) }}   # 70% old, 30% new
```

The predicted minutes are then `learned_minutes / current_speed`, plus a user safety
buffer. That feeds the feasibility check ("can this lawn still finish before the
deadline?").

### 4. The dispatcher — the one decision, asked every minute

The heart of it. Every minute (and on VPD-ready edges), if the mower is free and
healthy and it's dry, start **one** lawn that is due + VPD-ready (or overdue) +
feasible before the finish-by. Front goes first (it has the gate and dries earlier);
the back follows after the front finishes and recharges.

```yaml
conditions:
  - condition: state          # only when we've opted into VPD control
    entity_id: input_boolean.mow_vpd_driven_live
    state: 'on'
  - condition: state          # mower actually free
    entity_id: binary_sensor.lawn_mower_active
    state: 'off'
  # Rain fail-safe: skip if wet, actively raining, OR rain data is unknown.
  - condition: template
    value_template: >-
      {% set veto = is_state('input_boolean.mow_rain_veto_enabled','on') %}
      {% set wet  = states('binary_sensor.mow_lawn_wet') in ['on','unknown','unavailable'] %}
      {{ (not (veto and wet)) and is_state('binary_sensor.mow_rain_significant','off') }}
```

Note **unknown rain data is treated as wet** — a sensor dropping offline should hold
the mow, not silently clear the veto. (The trade-off: an offline rain sensor *can*
falsely block a mow. We pair it with a "held — here's why" notification so it never
blocks *silently*.)

Then each lawn's go/no-go folds due + ready + feasible + earliest-floor together:

```yaml
f_go: >-
  {% set feas = now() <= f_latest %}                 # finishes before deadline?
  {% set earliest_ok = now() >= today_at(states('input_datetime.mow_earliest_front')) %}
  {{ due and gate and earliest_ok
     and ( (ready and feas) or (backstop and over and ceiling and feas) ) }}
```

The `backstop` branch is an anti-overgrowth safety net: if VPD *never* crosses on a
cool damp day but the lawn is genuinely overdue, mow it at the last feasible moment
anyway — still obeying the rain veto and the earliest-floor.

### 5. Restart-resilient "a mow is in progress" flag

When the dispatcher launches a lawn it sets `input_boolean.mow_vpd_mowing` and stamps
the stop-time *immediately* (re-stamped accurately on real completion). That, plus a
watchdog that clears the flag if the mower isn't actually mowing/returning/docking for
a few minutes, prevents both **double-dispatch** and a **stuck flag** across restarts
or a dispatch that never really started.

---

## Rolling it out safely (how we did it)

The reference file is **additive and revertible** — delete it and restart to remove
it entirely. We also kept it **behind a master switch** (`input_boolean.mow_vpd_driven_live`):
until you flip that on, an older, simpler launcher stays in charge and everything here
is read-only/observational. That let us watch the "today's plan" preview sensor for a
couple of weeks and confirm it would have made the same calls we would, before handing
it the wheel. If you build something like this, we'd gently suggest the same: run it in
preview, compare, *then* go live.

---

## What to take from this

You almost certainly won't want our exact thresholds or two-lawn layout. The parts
worth stealing are the *patterns*:

- **VPD as the go-signal, rain as the veto** — different jobs, both needed.
- **Persist the sustain clock** (timestamp), don't use a `delay_on` that resets on
  restart.
- **Fail safe on missing data** — unknown VPD → not ready; unknown rain → treat as wet.
- **Learn durations** instead of hardcoding, so "finish by" stays honest.
- **One small decision asked repeatedly** beats one big brittle plan — complex
  behaviour (both lawns, split days) emerges for free.
- **Ship it behind a switch**, in preview, and keep it deletable.

Adapt freely. If something here is unclear, the reference YAML has the full comments.
