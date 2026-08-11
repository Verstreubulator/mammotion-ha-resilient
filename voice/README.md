# Optional voice layer (Home Assistant Assist)

This folder is the **optional cap** on the mammotion-ha-resilient stack. It adds
hands-free voice control and short spoken confirmations on top of the core and
orchestration tiers. **You do not need any of this** — the core status sensors
and the orchestration scripts run perfectly with no voice at all. Skip this
folder entirely if you don't want voice.

None of this would exist without the community integration
[`mikey0000/Mammotion-HA`](https://github.com/mikey0000/Mammotion-HA) and the
underlying `pymammotion` library. Everything here sits on top of the entities
that integration provides. Sincere thanks to its maintainers and contributors —
supporting a closed, changing mower protocol for free is hard work, and we are
grateful.

## What this layer adds

- **Sentence triggers** (`intents/mower.yaml`) — maps phrases like *"mow the
  front yard"* or *"stop the mower"* to intent names.
- **Intent handlers** (`intent_scripts.yaml`) — for each intent, an immediate
  spoken reply computed from the real mower status, a sanity pre-check, and a
  hand-off to the matching orchestration script.

The four intents:

| Say | Intent | Launches |
| --- | --- | --- |
| "mow the front yard" | `MowFrontYard` | `script.mow_the_front_yard` |
| "mow the backyard" | `MowBackyard` | `script.mow_the_backyard` |
| "mow all the lawns" | `MowAllLawns` | `script.mow_the_yard` |
| "cancel the mow" / "dock the mower" | `CancelMowing` | `script.mower_cancel_and_dock` |

## What it depends on (the lower tiers must already work)

- **Core status tier** — `sensor.mower_status`,
  `binary_sensor.mammotion_integration_healthy`,
  `input_boolean.mammotion_intentionally_disabled`,
  `input_boolean.mow_yard_active`, plus the integration-provided
  `sensor.<mower>_battery` and `lawn_mower.<mower>`.
- **Orchestration tier** — `script.mow_the_front_yard`,
  `script.mow_the_backyard`, `script.mow_the_yard`,
  `script.mower_cancel_and_dock`.

If those entities don't exist in your install, install the core and
orchestration packages first. The handlers here only *read* the status sensors
and *launch* the scripts; they never drive the mower directly.

## Two design principles worth borrowing

These are the ideas we found most useful. Adopt them however you like — the
exact wiring below is just what happened to work for us.

### 1. The immediate REPLY is separate from background ANNOUNCEMENTS

There are two very different kinds of speech:

- **The reply** — the quick "got it" the satellite speaks the instant you give
  a command. That lives in `speech.text` in `intent_scripts.yaml`, and we
  compute it from the real status sensors so it's honest: *"Battery at 54
  percent. Minimum is 70."*, *"Mower offline. Try again shortly."*, or
  *"Starting front yard."* You spoke to it, so a reply now is expected and
  welcome.
- **The announcement** — later progress messages like *"front yard finished"*.
  Those are the **orchestration scripts'** job, not this layer's. Keeping them
  separate means the reply stays instant and the announcement can be gated
  independently (next point).

### 2. Background announcements route through a quiet-hours / DND gate

An immediate reply is fine any time — you just asked for it. A *background*
announcement that fires on its own is different: spoken audio in the middle of
the night is exactly the kind of thing that annoys a household. So route your
background announcements through a quiet-hours / Do-Not-Disturb gate:

- During quiet hours, **suppress the spoken audio** but **still post the
  notification** (a phone push, a persistent notification, a logbook entry) so
  nothing is lost.
- Outside quiet hours, speak normally.

This layer deliberately does **not** ship that gate — it's a personal policy and
belongs in your own `notify` / announce helper. The intent handlers here only
produce the immediate reply; wire your announce/notify plumbing and its DND gate
yourself. (In our install the mow scripts call a single `script.announce` helper
that owns the quiet-hours decision; yours can be anything from a template script
to a simple `notify` service.)

## Satellite hardware

We used **Home Assistant Voice PE (VPE)** satellites, but that's just what we
happened to have. Any Assist pipeline works — a Voice PE, an Atom Echo, a phone
running the HA app, the web Assist dialog, whatever. Nothing here is VPE-specific.

### Optional: reply on the satellite you spoke to

If you have several satellites and want the *finish* announcement to come back
on the same speaker you gave the command to, the handlers stash the triggering
satellite into a helper (`input_text.mower_source_satellite`) that the
orchestration script can read later. This is entirely optional:

- If you don't care which speaker answers, delete the `source_satellite`
  variable block and the `input_text.set_value` step in each handler.
- If you do want it, create the helper and point the resolution template's
  fallback (`assist_satellite.YOUR_DEFAULT_SATELLITE`) at one of your own
  satellites:

  ```yaml
  input_text:
    mower_source_satellite:
      name: Mower source satellite
  ```

## Install

1. **Rename the mower slug.** Every `luba_3` in `intent_scripts.yaml`
   (e.g. `sensor.luba_3_battery`, `lawn_mower.luba_3`) came from our mower's
   device name (a Mammotion Luba 3). Replace it with your own mower's entity
   slug — find it in **Developer Tools → States**.

2. **Sentences** — `intents/mower.yaml`. Put it where your Assist config reads
   custom sentences and reference it from `configuration.yaml`, e.g.:

   ```yaml
   # configuration.yaml
   conversation:
     intents: !include_dir_merge_named intents/
   ```

   (or copy the file to `config/custom_sentences/en/mower.yaml` if you use that
   convention). This file **hot-reloads** — no restart needed. After editing,
   use **Developer Tools → YAML → Reload custom sentences** (or reload the
   conversation config).

3. **Intent handlers** — `intent_scripts.yaml`. Register it under
   `intent_script:` in `configuration.yaml`, either inline or as an include:

   ```yaml
   # configuration.yaml
   intent_script: !include intent_scripts.yaml
   ```

   > ⚠️ **`intent_script:` has NO hot reload.** After adding or editing this
   > file you **must fully restart Home Assistant**. (Only the sentences file
   > in step 2 reloads live.)

4. **Optional satellite helper** — create `input_text.mower_source_satellite`
   (see above) only if you use the reply-on-triggering-satellite feature.

5. **Test.** Say *"mow the front yard"* to any Assist entry point and confirm
   you get the spoken reply and the mow starts. If the reply says *"Mower
   offline"* or *"Battery at N percent"*, that's the pre-check doing its job —
   check the core status sensors.
