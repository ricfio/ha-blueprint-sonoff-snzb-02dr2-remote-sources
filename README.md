# SONOFF SNZB-02DR2 (Zigbee2MQTT) - Remote Sources

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fricfio%2Fha-blueprint-sonoff-snzb-02dr2-remote-sources%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Fsonoff-snzb-02dr2-remote-sources.yaml)

A Home Assistant blueprint that sends the temperature and/or humidity of up to two
other sensors to the [**SONOFF SNZB-02DR2 (AirGuard TH)**](https://sonoff.tech/products/sonoff-airguard-th-3-6-lcd-zigbee-temperature-and-humidity-sensor-snzb-02dr2),
so they are shown on its display (for example the reading of an outdoor sensor).

<!-- Add a photo or screenshot of the display here, e.g.:
![Display showing EXT1 and EXT2](docs/display.jpg) -->

## Features

- Two independent remote slots, shown on the display as **EXT1** and **EXT2**, usable together
- For each slot: temperature, humidity, or both (every source is optional)
- Values are validated before being sent: numeric, supported unit, within range, and not stale
- Fahrenheit and Kelvin temperature sources are converted to °C; unknown units are rejected
- Changes to a source's value or temperature unit trigger an update; unrelated attribute changes are ignored
- Configurable fallback for each item when a source is not valid
- Periodic refresh and a restartable source-change debounce, so bursts send the latest reading

## Requirements

- Home Assistant 2024.10 or later
- MQTT integration enabled in Home Assistant
- Zigbee2MQTT exposing the `source_N_*` properties for the display
- SONOFF SNZB-02DR2 firmware 1.0.5 or later (first version with the two remote sources)

ZHA is not supported: the blueprint writes directly to the Zigbee2MQTT `set` topic.

### Tested with

- SONOFF SNZB-02DR2, firmware 1.0.5

Other firmware versions have not been tested.

## Installation

1. Click the **Import blueprint** button above, or import this URL from
   *Settings → Automations & scenes → Blueprints → Import blueprint*:

   ```
   https://github.com/ricfio/ha-blueprint-sonoff-snzb-02dr2-remote-sources/blob/main/blueprints/automation/sonoff-snzb-02dr2-remote-sources.yaml
   ```

Manual alternative: copy `blueprints/automation/sonoff-snzb-02dr2-remote-sources.yaml`
into `<config>/blueprints/automation/` (any subfolder works) and reload.

2. Open the blueprint and click **Create automation**.
3. Enter the Zigbee2MQTT name of the display and pick the source sensors for EXT1 and/or EXT2.
4. Save.

## Configuration

### Display

| Input | Default | Description |
|---|---|---|
| Zigbee2MQTT name of the display | – | The `friendly_name` of the SNZB-02DR2 in Zigbee2MQTT, case-sensitive |
| MQTT base topic | `zigbee2mqtt` | Change only if you changed the Zigbee2MQTT `base_topic` |

### EXT1 and EXT2

| Input | Default | Description |
|---|---|---|
| Temperature sensor | empty | Leave empty to send nothing |
| Temperature fallback | default | What to write if the temperature source is not valid |
| Humidity sensor | empty | Leave empty to send nothing |
| Humidity fallback | default | What to write if the humidity source is not valid |

### Update frequency

| Input | Default | Description |
|---|---|---|
| Periodic refresh | every 30 minutes | Resends all values at regular intervals, even without changes |
| Source update debounce | 60 s | Waits for a quiet period after source changes, then sends the latest state. Periodic refresh and startup run immediately |

### Safety and fallback

| Input | Default | Description |
|---|---|---|
| Maximum age of a source reading | 120 min | Older readings are considered not valid. `0` disables the check |
| Default fallback | Do nothing | Applied to every item set to "Use the default fallback" |
| Restore "online" | off | Writes `online` before every valid configured value; safe to enable even when no fallback was previously sent |

### Advanced

| Input | Default | Description |
|---|---|---|
| Set `remote_source_status` to "using" | on | Required to enable remote sources on firmware 1.0.5 and later; sent before the source values |
| Pause between writes | 2 s | Delay between two consecutive MQTT messages |

## How it works

The automation runs when:

- the **state** or `unit_of_measurement` of a configured source changes (unrelated attribute-only changes are ignored),
- the **periodic refresh** fires,
- **Home Assistant starts**.

On every run, all configured sources are evaluated, so a change on any of them refreshes the
whole display. Source changes restart a debounce timer; when the source has been quiet for
the selected interval, the automation evaluates the current states. This coalesces rapid
updates without dropping the final useful reading. Periodic refresh and Home Assistant
startup bypass the debounce. A new source change during a publish sequence restarts the run;
the new run evaluates and sends the latest state.

Each value is published in its own MQTT message to
`<base_topic>/<display name>/set` (`source_N_temperature` / `source_N_humidity`), with
a pause between messages. The order for an item is its optional `online` state followed by
its value; invalid items receive only the selected fallback state. `remote_source_status`
is sent first; it is enabled by default. QoS 1 and `retain: false` are used for every command.

A reading is sent only if it is **valid**:

- the state is numeric and not `unknown`/`unavailable`
- temperature uses °C/℃, °F/℉ or K and converts to -20…60 °C before rounding to one decimal place for transmission
- humidity uses `%` and is within 0…100 % (100 is sent as the device maximum, 99.9)
- the state object's `last_reported` time is within the configured maximum age; `0` disables
  this check. Unrelated attribute-only updates do not trigger a run, but are considered when a run occurs.

### Fallback

When a configured source is not valid, the fallback decides what is written for that item
only (for example EXT1 temperature):

| Option | Effect |
|---|---|
| Do nothing | Nothing is written; the display keeps its last value |
| Mark offline | Writes `offline` to `source_N_*_state` |
| Unbind | Writes `unbound` to `source_N_*_state` |

Each item can use the default fallback or its own. How the display renders these states
depends on the firmware. When a valid reading returns, enabling **Restore "online"** writes
`online` immediately before the measurement; Zigbee2MQTT documents that this state-only
update does not overwrite the stored measurement. The default fallback `Do nothing` leaves
the last displayed reading/state untouched.

## Known limitations

- The automation uses restart mode. A new source update cancels a pending debounce or
  publish sequence and starts again with current states; keep the pause between writes modest.
- The display is a battery-powered sleepy device. According to the Zigbee2MQTT
  documentation, new values are applied at its next check-in (the screen refreshes
  from the remote sources every 30 minutes) or immediately after briefly pressing the
  button on its back.

## Troubleshooting

- **Nothing changes on the display**
  - Check that the display name matches the Zigbee2MQTT `friendly_name` exactly
 (case-sensitive) and that the base topic is correct.
  - Check in Zigbee2MQTT that the device exposes the `source_N_*` properties.
  - Look at the automation trace to see which messages were published.
  - Verify that *Set `remote_source_status` to "using"* is enabled in the Advanced section; it is on by default and required for remote sources.
- **A slot stays empty even though the sensor works**
  - The reading may not be valid: out of range, not numeric, or older than the
 configured maximum age.

## Links

- [SONOFF AirGuard TH (SNZB-02DR2) product page](https://sonoff.tech/products/sonoff-airguard-th-3-6-lcd-zigbee-temperature-and-humidity-sensor-snzb-02dr2)
- [Zigbee2MQTT device page](https://www.zigbee2mqtt.io/devices/SNZB-02DR2.html)
- [SONOFF documentation, including the official Home Assistant blueprint](https://help.sonoff.tech/docs/SNZB02DR2#Home_Assistant)

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for release history.

## License

[MIT](LICENCE)

This project is not affiliated with or endorsed by SONOFF or ITEAD.
