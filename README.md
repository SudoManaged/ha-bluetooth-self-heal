# Bluetooth self-heal for Home Assistant

[![Validate](https://github.com/SudoManaged/ha-bluetooth-self-heal/actions/workflows/validate.yml/badge.svg)](https://github.com/SudoManaged/ha-bluetooth-self-heal/actions/workflows/validate.yml)
![Home Assistant 2024.11+](https://img.shields.io/badge/Home%20Assistant-2024.11%2B-41BDF5)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

Bring your Bluetooth sensors back automatically when Home Assistant's Bluetooth adapter stops delivering readings.

A free, open-source Home Assistant blueprint by SudoManaged. It runs entirely on your own Home Assistant: no
add-ons, no custom integrations, no cloud services, no accounts. It started as an automation that runs every day
on our own Home Assistant, and was then generalized and tested for release.

## Contents

- [Requirements and tested versions](#requirements-and-tested-versions)
- [Installation](#installation)
- [What it is for](#what-it-is-for)
- [How it works](#how-it-works)
- [Setting it up](#setting-it-up)
- [Settings](#settings)
- [What a recovery looks like](#what-a-recovery-looks-like)
- [Picking good sensors](#picking-good-sensors)
- [Several adapters and Bluetooth proxies](#several-adapters-and-bluetooth-proxies)
- [Troubleshooting](#troubleshooting)
- [Frequently asked questions](#frequently-asked-questions)
- [Limitations](#limitations)
- [Updating and removing](#updating-and-removing)
- [Security](#security)
- [Reporting problems and contributing](#reporting-problems-and-contributing)
- [More free Home Assistant tools from SudoManaged](#more-free-home-assistant-tools-from-sudomanaged)
- [Changelog and license](#changelog-and-license)

## Requirements and tested versions

- **Home Assistant 2024.11 or newer.** The blueprint declares this, and Home Assistant refuses to import it from
  the UI on an older release.
- Home Assistant's **Bluetooth** integration with a **local adapter** (built in or USB), and two or more Bluetooth
  sensors heard through it.
- The **logbook** integration (part of Home Assistant's default configuration).
- It passes Home Assistant's configuration check on **2024.11.0**, **2026.3.3** and **2026.11.0**, and was run
  live on all three with simulated sensors: all sensors unavailable, a reload that does not help (notification),
  a retry after the cooldown that does help (notification dismissed), and the restart option (Home Assistant
  restarted).
- Every change to this repository is checked automatically against Home Assistant 2024.11.0 and the current
  release (see [`.github/workflows/validate.yml`](.github/workflows/validate.yml)).

## Installation

[![Import the Bluetooth self-heal blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FSudoManaged%2Fha-bluetooth-self-heal%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fsudomanaged%2Fbluetooth_self_heal.yaml)

**With the button:** click **Import blueprint** above. It opens your own Home Assistant (through
my.home-assistant.io) with the blueprint ready to import.

**By hand:** copy [`blueprints/automation/sudomanaged/bluetooth_self_heal.yaml`](blueprints/automation/sudomanaged/bluetooth_self_heal.yaml)
to `/config/blueprints/automation/sudomanaged/bluetooth_self_heal.yaml` in your Home Assistant configuration
folder, then reload automations (**Developer tools, YAML, Automations**) or restart Home Assistant.

Then follow [Setting it up](#setting-it-up).

## What it is for

Bluetooth thermometers, humidity sensors and plant sensors depend on Home Assistant's Bluetooth adapter. USB
Bluetooth adapters sometimes freeze or drop off the USB bus, and Home Assistant does not always recover from that
on its own. Nothing errors loudly: your sensors simply go unavailable, and automations that depend on them (a
dehumidifier, a freezer alarm) stop working until someone notices.

This blueprint notices for you and fixes the common case on its own.

## How it works

1. **Watching.** Home Assistant marks a Bluetooth sensor unavailable when it has not heard from it for a while:
   after 15 minutes of silence, checked every 5 minutes, so up to about 20 minutes; sooner for devices whose
   broadcast interval it has learned. The blueprint looks at your chosen sensors as soon as one has been unavailable for **Unavailable for**
   minutes, and again every 5 minutes.
2. **Deciding.** It acts only when **every** chosen sensor has been unavailable (or unknown) for at least
   **Unavailable for** minutes, and the cooldown has passed. One sensor with a flat battery does not set it off;
   the adapter going quiet does.
3. **Fixing.** It reloads the Bluetooth adapter you picked, which makes Home Assistant open the adapter again
   (it finds it by its address, even if the system gave it a new name such as `hci1`). If the reload itself fails
   (for example the entry was removed), it carries on to the next step.
4. **Checking.** It waits up to **Wait for the sensors to come back** seconds for any chosen sensor to become
   available again, which happens when Home Assistant receives a new broadcast from it.
5. **Escalating.** If a sensor comes back, it logs that the reload worked and removes this automation's earlier
   "still unavailable" notification. If not, it either restarts Home Assistant (if you turned that on) or creates
   a notification, titled with the automation's name, asking you to check the adapter.
6. **Cooling down.** It does not run again within **Cooldown** minutes. While the sensors stay unavailable, it
   tries again at the first 5-minute check once the cooldown has passed.

Every step is written to the logbook under the automation's name, attached to the automation.

## Setting it up

1. Open **Settings, Devices & services, Bluetooth** and note your local adapter (named after its MAC address or
   model). Since Home Assistant 2025.2, Bluetooth proxies (ESPHome, Shelly) are listed there too; this blueprint
   is for the local adapter.
2. Pick two or more sensors heard through that adapter (see [Picking good sensors](#picking-good-sensors)).
3. Go to **Settings, Automations & scenes, Blueprints**, open **Bluetooth self-heal** and click **Create
   automation**.
4. Choose the sensors and the adapter, keep the other settings for now, and save.

Or in YAML (`automations.yaml`):

```yaml
- id: bluetooth_self_heal
  alias: Bluetooth self-heal
  use_blueprint:
    path: sudomanaged/bluetooth_self_heal.yaml
    input:
      sensors:
        - sensor.living_room_thermometer_temperature
        - sensor.bedroom_thermometer_temperature
      bluetooth_entry: YOUR_BLUETOOTH_CONFIG_ENTRY_ID
```

In YAML, `bluetooth_entry` is the adapter's config entry ID. The easiest way to get it is to create the automation
in the UI once and copy the ID from its YAML view.

## Settings

| Setting | Default | Range | Explanation |
|---|---|---|---|
| **Bluetooth sensors to watch** | (required) | | Two or more sensors heard through the adapter below. All of them must be unavailable before anything happens. |
| **Bluetooth adapter** | (required) | | The local adapter's entry under **Settings, Devices & services, Bluetooth**. With several adapters, pick the one these sensors use. Not a Bluetooth proxy. |
| **Unavailable for (minutes)** | 2 | 1 to 60 | How long every sensor must have been unavailable before acting. This comes on top of the time Home Assistant takes to mark them unavailable (up to about 20 minutes). |
| **Wait for the sensors to come back (seconds)** | 90 | 15 to 600 | How long to wait after the reload for any sensor to become available again. |
| **Restart Home Assistant if the reload does not help** | Off | | On: restart as the last resort, at most once per cooldown. Off: you get a notification instead and decide yourself. |
| **Cooldown (minutes)** | 30 | 5 to 1440 | Minimum time between runs. With restarts on, use several hours (for example 360), so a dead adapter does not restart Home Assistant every half hour. |

## What a recovery looks like

Open **Logbook** in the sidebar and filter it by the automation. Next to "triggered by" lines, a successful
recovery shows these entries (the logbook lists the newest first; the name is your automation's name):

```
Bluetooth self-heal   All watched Bluetooth sensors are unavailable; reloading the Bluetooth adapter
Bluetooth self-heal   The reload brought the sensors back
```

and one that needed more help:

```
Bluetooth self-heal   All watched Bluetooth sensors are unavailable; reloading the Bluetooth adapter
Bluetooth self-heal   The reload did not bring the sensors back; restarting Home Assistant
```

(or "The reload did not bring the sensors back", with a notification, when restarts are turned off).

The automation also shows **Traces**, but the check every 5 minutes leaves a trace each time, and Home Assistant
keeps only the last 5 by default, so the trace of a repair is gone within about 25 minutes. The logbook keeps the
record. To keep more traces, add `trace:` with `stored_traces: 50` to the automation in YAML.

## Picking good sensors

- **Two or more sensors heard through the same adapter.** With one sensor, a flat battery looks exactly like a
  dead adapter. With two, the blueprint only acts when both have been unavailable for the set time.
- **Sensors that broadcast regularly** (every few seconds to a few minutes), such as Bluetooth thermometers and
  humidity sensors (Govee, Xiaomi, Inkbird, SwitchBot and similar). How often their *values* change does not
  matter: availability is what counts.
- **Sensors that are reliably in range**, with healthy batteries. A sensor that drops out on its own now and then
  is fine as long as the others stay up, but if all of them are flaky, the blueprint will reload the adapter for
  nothing.

## Several adapters and Bluetooth proxies

- **More than one local adapter:** each adapter is its own entry under Bluetooth. Create one automation from this
  blueprint per adapter, each watching sensors heard through that adapter.
- **Bluetooth proxies (ESPHome, Shelly):** sensors heard through a proxy do not depend on the local adapter, and
  reloading a proxy's entry does nothing useful. Watch only sensors heard through the local adapter, and fix an
  offline proxy at the proxy.
- **Which adapter hears a sensor:** Home Assistant's Bluetooth advertisement monitor (see the
  [Bluetooth integration documentation](https://www.home-assistant.io/integrations/bluetooth/)) lists each
  device with the adapter or proxy that heard it last.

## Troubleshooting

- **It never runs.** Check that *all* chosen sensors were unavailable at the same time for longer than
  **Unavailable for**, and that the cooldown had passed. The automation's traces show why a check stopped (look at
  the most recent ones, see [What a recovery looks like](#what-a-recovery-looks-like)).
- **It runs although Bluetooth is fine.** All chosen sensors went unavailable together for another reason, for
  example they are all far from the adapter or share a power source. Pick sensors that are reliably in range.
- **The reload never helps.** The adapter is probably gone at the USB level. Turn on the restart option, or
  unplug and replug the adapter. If it happens often, a USB extension cable (to move the adapter away from USB 3
  ports and the computer) or a Bluetooth proxy usually fixes the root cause.

## Frequently asked questions

**Will it restart Home Assistant on its own?** Only if you turn on **Restart Home Assistant if the reload does not
help**. By default it never restarts anything; it reloads the adapter and, if that is not enough, tells you.

**Can it keep restarting Home Assistant?** With restarts on and an adapter that stays dead, it restarts Home
Assistant at most once per cooldown, every cooldown, until the sensors come back. That is why a cooldown of
several hours is recommended with restarts on.

**Does it work with Bluetooth sensors that use their own integration?** Yes, as long as they are heard through
Home Assistant's Bluetooth integration, which is how most Bluetooth sensor integrations work.

**Does it need internet access?** No. Everything happens inside Home Assistant.

## Limitations

- A reload cannot revive an adapter that is physically dead or missing; that needs a restart or a replug.
- It watches sensors, not the adapter itself, and only acts once Home Assistant has marked them unavailable, which
  can take up to about 20 minutes after the adapter stops.
- A sensor that stays unavailable or unknown on its own (dead battery, out of range) counts as down all the time,
  so a two-sensor setup then acts on the other sensor alone. Fix or remove such a sensor from the list.
- Sensors that are unavailable from the moment Home Assistant starts (for example after a restart with a dead
  adapter) are picked up by the check every 5 minutes.

## Updating and removing

- **Updating:** in **Settings, Automations & scenes, Blueprints**, open the blueprint's menu and choose
  **Re-import blueprint** (or replace the file by hand and reload automations). Your automations keep their
  settings.
- **Removing:** delete the automations that use the blueprint, then delete the blueprint. Nothing else is
  installed.

## Security

- The blueprint makes no network calls and opens nothing to the network. Its only actions are local: reloading
  the Bluetooth adapter, writing to the logbook, a Home Assistant notification, and (only if you turn it on)
  restarting Home Assistant.
- To report a security problem privately, use **Report a vulnerability** on this repository's
  [Security tab](../../security). Please do not open a public issue for security problems. See
  [SECURITY.md](SECURITY.md).

## Reporting problems and contributing

- **Something does not work:** open an [issue](../../issues) with your Home Assistant version, your adapter, what
  you expected and what happened. The automation's logbook entries and a trace help a lot; remove anything
  private from them first.
- **Improvements:** pull requests are welcome. Keep the blueprint self-contained (no custom integrations) and
  documented in this README.

## More free Home Assistant tools from SudoManaged

- [Webhook alerts](https://github.com/SudoManaged/ha-webhook-alerts)
- [Yealink desk phone call log](https://github.com/SudoManaged/ha-yealink-call-log)
- [Dehumidifier kit](https://github.com/SudoManaged/ha-dehumidifier-kit)
- [Internet failover alerts](https://github.com/SudoManaged/ha-internet-failover)

## Changelog and license

See [CHANGELOG.md](CHANGELOG.md). Released under the [MIT License](LICENSE). Copyright (c) 2026 SudoManaged.
