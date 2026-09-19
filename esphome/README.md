# ESP32 relay scheduler (ESPHome)

[`relay-scheduler.yaml`](relay-scheduler.yaml) turns a relay on and off on a
daily schedule, by hand, or both. Schedule times, days and the on/off switch
are runtime entities, so they can be changed from Home Assistant or the
device's own web page without re-flashing.

## Hardware

| Part | Default pin | Notes |
| --- | --- | --- |
| Relay module IN | `GPIO26` | Set `relay_inverted: "true"` for active-LOW boards (most cheap ones are). |
| Push button | `GPIO0` | Optional, momentary to GND. `GPIO0` is the on-board BOOT button and a strapping pin — fine for a momentary press, but don't hold it at boot. Delete the `binary_sensor` block if unused. |

Power the relay module from 5V (or its own supply) and tie its ground to the
ESP32's ground. Mains wiring is on the relay's output side — switch the live
conductor, not neutral, and keep it isolated from the low-voltage side.

## Setup

```bash
cp esphome/secrets.yaml.example esphome/secrets.yaml   # then fill it in
esphome run esphome/relay-scheduler.yaml
```

Adjust the `substitutions:` block at the top of the YAML for your pins and
timezone before the first flash.

## Entities

| Entity | What it does |
| --- | --- |
| **Relay** | The relay. Toggle it any time — this is the manual control. |
| **Button** | Physical button; a press toggles the relay. |
| **Schedule enabled** | Off = manual only; the schedule never touches the relay. |
| **On hour / On minute** | Time the relay switches on. |
| **Off hour / Off minute** | Time the relay switches off. |
| **Schedule days** | `Every day`, `Weekdays` (Mon–Fri) or `Weekends`. |
| **Resync to schedule** | Applies the schedule immediately — use it to drop a manual override. |
| **Next scheduled change** | Diagnostic text showing the next switch time. |

## How manual and scheduled control interact

The relay is a plain switch, so a manual toggle always wins *at that moment*.
The schedule then reasserts itself at the next on or off time. Concretely,
with a 07:30–22:00 window: switch the relay off by hand at 09:00 and it stays
off until 07:30 the next day, when the schedule turns it back on.

Three things re-apply the schedule immediately rather than waiting for the
next boundary: boot, the first NTP sync, and the **Resync to schedule**
button. Resync works out whether *now* is inside the window — including
overnight windows such as on 22:00 / off 06:00, where the day filter is
applied to the current day — and sets the relay accordingly. That way a power
cut in the middle of a window doesn't leave the relay stuck in the wrong
state.

Setting the on and off times equal disables the schedule's switching without
turning the schedule off.

## Extras

**Auto-off after a manual turn-on** — add a timeout so a hand-started run
can't be left on forever:

```yaml
switch:
  - platform: gpio
    id: relay
    # ...existing config...
    on_turn_on:
      - delay: 30min
      - switch.turn_off: relay
```

**A second daily window** — add another `on_time` trigger with fixed times:

```yaml
time:
  - platform: sntp
    id: esp_time
    on_time:
      - hours: 12
        minutes: 0
        seconds: 0
        then:
          - switch.turn_on: relay
```

**No Home Assistant?** The device serves its own page at
`http://<device-ip>/` with every entity above, and raises a fallback Wi-Fi
access point if it can't join your network.
