# Optional: gating a zone on a gate or door sensor

In our own yard, one lawn sits behind a physical garden gate the mower has to drive through. So our
real flow checks a gate sensor before mowing that zone. We **left this out of the shipped scripts** on
purpose — it's specific to our yard, and carrying it inside the mow script makes that script noisier
for the majority of people who don't have a gate.

If a zone of yours *is* behind a gate or a door, here's the small pattern to add it back. It's a
**no-wait check**: if the gate isn't open, don't mow that zone — rather than blocking for minutes
hoping someone opens it.

## What you need

A binary sensor that reads `on` when the gate/door is open — a reed/contact sensor, a cover entity, a
camera-derived sensor, whatever you have. We'll call it `binary_sensor.your_gate` below.

## The single-zone version (simplest)

Add this near the top of the mow script for the gated zone (e.g. inside `mow_the_front_yard`), after
the integration/ready gates and before selecting the zone:

```yaml
# OPTIONAL garden-gate check — only mow if the gate is open. No wait: if it's
# closed we skip rather than block. Point this at your own gate/contact sensor.
- if:
    - condition: template
      value_template: "{{ states('binary_sensor.your_gate') != 'on' }}"
  then:
    - action: persistent_notification.create
      data:
        title: "Mow skipped — gate closed"
        message: "The gate was closed, so the front yard wasn't mowed. Open it and try again."
    - stop: "Gate closed — skipping this zone"
```

That's the whole idea. The mower simply won't leave the dock for that zone unless the gate reads open.

## If you use the whole-yard orchestration

The shipped `mow_the_yard` orchestration runs a straight **front → back** flow. If you want a gate to
influence the *order* — mow the backyard first when the front gate is closed, then come back and retry
the front once someone opens it — you'll need to reintroduce a couple of phases. In our private setup
that path looked like:

```
front gate closed  ->  mow backyard first  ->  recharge  ->  re-check the gate once  ->
    open?  mow the front  |  still closed?  end and notify "open the gate and run Mow the Front Yard"
```

We deliberately don't ship that phase machine here because it roughly doubles the orchestration's
complexity for a very yard-specific feature, and a phase machine is easy to get subtly wrong. If you
want it, the building blocks are all in
[`packages/mammotion_orchestration.yaml`](../packages/mammotion_orchestration.yaml): add the extra
phases to `input_select.mow_yard_phase`, branch on the gate check in the "after front" advance
automation, and add a one-shot "retry front" phase after the backyard finishes. Start from the
single-zone check above and grow it only if you actually need the reorder.

## Why no-wait, not wait

An earlier version of ours waited up to 5 minutes for the gate to open. It mostly just delayed the
inevitable and complicated the flow. A no-wait check is simpler, never leaves the mower half-committed,
and pairs naturally with "try again later." If you genuinely want a wait, a `wait_template` on the gate
sensor with a modest timeout works — but treat the timeout as a backstop, not the control mechanism
(see [self-recovery.md](self-recovery.md#timeouts-backstops-not-control-mechanisms)).
