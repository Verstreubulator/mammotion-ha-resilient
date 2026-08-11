# The zone-switch fix (issue #604)

This is the single most useful idea in the repo. If you build any automation that selects mowing
zones, you need this or something like it.

## The problem

The Mammotion integration exposes each mowing zone/area as a **`switch`** entity. But it
**re-derives those switch `entity_id`s every time the map is synced or an area is edited** — it
appends an incrementing numeric suffix and leaves the previous entity orphaned in the `unavailable`
state.

This isn't a rare event. It fires on *any* structural map change, not just a full re-add. A single
example from our install: recreating one "Back Lawn" area turned the live switches into

```
switch.…_area_back_lawn_3
switch.…_area_front_lawn_w_4
switch.…_area_front_lawn_e_2
```

…and left six older `…_area_*` switches sitting `unavailable` in the registry.

This is tracked upstream as [mikey0000/Mammotion-HA #604](https://github.com/mikey0000/Mammotion-HA/issues/604).
It's marked closed, but in the versions we run the entity-id churn still happens. **We don't say this
to fault anyone** — the `unique_id`s the integration assigns are stable zone hashes; it's Home
Assistant that derives a fresh visible `entity_id` when the area is re-published. Untangling that for
a closed, changing mower protocol is genuinely hard.

## Why it bites so badly

The failure is **silent**. An automation that hardcoded `switch.…_area_back_lawn` will:

1. call `switch.turn_on` on an entity that now 404s,
2. get no error the user ever sees,
3. call `lawn_mower.start_mowing` — which "succeeds",
4. and the mower **never moves**, because no zone was actually selected.

You get a confident "mowing the back yard now" and a mower sitting on its dock. (See
[lessons-learned.md](lessons-learned.md) §10.2.)

## The fix: resolve by substring + available-filter

**Never hardcode the zone `entity_id`. Resolve it at runtime** by matching a *stable substring* of
the name and filtering to the one that's actually available.

The key insight is the second filter. Matching the substring alone isn't enough — the orphaned
`unavailable` entities still match it. You have to drop anything that isn't `on`/`off`:

```jinja
{% set m = states.switch
     | selectattr('entity_id', 'search', 'area_front_lawn_w')
     | selectattr('state', 'in', ['on', 'off'])
     | map(attribute='entity_id') | list %}
{{ m[0] if m else '' }}
```

- `selectattr('entity_id', 'search', 'area_front_lawn_w')` — unanchored substring match, so it
  survives the `_2` / `_3` / `_4` suffix churn.
- `selectattr('state', 'in', ['on','off'])` — **this is the essential part.** It drops the
  `unavailable` orphans so only the live switch survives. (An anchored `…_w$` regex would instead
  risk matching a ghost — that was a real bug we hit; see §10.28.)

## Centralize it: the three display proxies

Rather than repeat that template in every script, dashboard card, and sensor, we put it in **one
place** — three read-only template `binary_sensor`s that mirror the live switch and expose the
resolved id as an attribute. This is [`packages/mammotion_core.yaml`](../packages/mammotion_core.yaml):

- `binary_sensor.mow_area_back_lawn`
- `binary_sensor.mow_area_front_lawn_w`
- `binary_sensor.mow_area_front_lawn_e`

Each one:
- **mirrors** its live switch's `on`/`off` state,
- exposes the live `entity_id` as a **`resolved_entity`** attribute (`''` when none is available),
- reports **`unavailable`** (honestly) when no live switch exists — mid-remap or zone truly gone,
  rather than a misleading `off`.

Then **every consumer reads the proxy, never the raw switch**:
- dashboards point their "Lawn Areas" card at the three proxies;
- the health gate checks each proxy is `in ['on','off']`;
- `sensor.mower_status` reads the proxy states for its `current_zone`;
- the mow scripts actuate via `state_attr('binary_sensor.mow_area_back_lawn', 'resolved_entity')`.

The result is **remap-proof**: the next re-enumeration can't break anything, because the proxy
resolves the new id and every consumer follows automatically, with zero intervention. We watched this
happen live — the back lawn churned `_3 → _4` mid-session and nothing downstream noticed.

## What you edit for your setup

Open `packages/mammotion_core.yaml` and change the **substrings** to match your own area names as they
appear in your switch entity_ids. If your areas are "Orchard" and "Side Strip", you'd match
`area_orchard` and `area_side_strip`, and rename the proxies to suit. That's the only zone-specific
edit — everything downstream keys off the proxies.

## A note for the maintainers

If a future integration release derives the visible `entity_id` from the stable zone `unique_id` (so
it no longer churns on map sync), this entire workaround becomes unnecessary and we'd happily delete
it. Until then, this pattern lets people build reliable zone automations today without waiting on an
upstream change. Offered with thanks, not as a complaint.
