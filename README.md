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
- Values are validated before being sent: numeric, within range, and not stale
- Fahrenheit and Kelvin sources are converted to °C
- Configurable fallback for each item when a source is not valid
- Periodic refresh and minimum interval between runs, to avoid flooding Zigbee

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
| Minimum interval between runs | 60 s | Runs triggered too soon after the previous one are skipped |

### Safety and fallback

| Input | Default | Description |
|---|---|---|
| Maximum age of a source reading | 120 min | Older readings are considered not valid. `0` disables the check |
| Default fallback | Do nothing | Applied to every item set to "Use the default fallback" |
| Restore "online" | off | Writes `online` before a valid value for items with an active fallback |

### Advanced

| Input | Default | Description |
|---|---|---|
| Also set `remote_source_status` to "using" | off | Enable only if the display does not show the values on its own |
| Pause between writes | 2 s | Delay between two consecutive MQTT messages |

## How it works

The automation runs when:

- the **state** of a configured source changes (attribute-only changes are ignored),
- the **periodic refresh** fires,
- **Home Assistant starts**.

On every run, all configured sources are sent, so a change on any of them refreshes the
whole display. Each value is published in its own MQTT message to
`<base_topic>/<display name>/set` (`source_N_temperature` / `source_N_humidity`), with
a pause in between. With the default 2 s pause and all four sources configured, a run
takes about 8 seconds; the optional messages (`remote_source_status`, `online`
restore) add more.

A reading is sent only if it is **valid**:

- the state is numeric
- temperature is between -20 and 60 °C, humidity between 0 and 100 % (humidity is
  sent capped at 99.9)
- the sensor has reported within the configured maximum age

### Fallback

When a configured source is not valid, the fallback decides what is written for that item
only (for example EXT1 temperature):

| Option | Effect |
|---|---|
| Do nothing | Nothing is written; the display keeps its last value |
| Mark offline | Writes `offline` to `source_N_*_state` |
| Unbind | Writes `unbound` to `source_N_*_state` |

Each item can use the default fallback or its own. How the display renders these states
depends on the firmware.

## Known limitations

- The automation runs in `single` mode and ignores new triggers while a run is in
  progress. Changes that arrive during a run are not processed immediately; the
  periodic refresh picks them up.
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
  - Try enabling *Also set remote_source_status to "using"* in the Advanced section.
- **A slot stays empty even though the sensor works**
  - The reading may not be valid: out of range, not numeric, or older than the
    configured maximum age.

## Links

- [SONOFF AirGuard TH (SNZB-02DR2) product page](https://sonoff.tech/products/sonoff-airguard-th-3-6-lcd-zigbee-temperature-and-humidity-sensor-snzb-02dr2)
- [Zigbee2MQTT device page](https://www.zigbee2mqtt.io/devices/SNZB-02DR2.html)
- [SONOFF documentation, including the official Home Assistant blueprint](https://help.sonoff.tech/docs/SNZB02DR2#Home_Assistant)

## Changelog

### 1.0.0

- First release.

## License

[MIT](LICENSE)

This project is not affiliated with or endorsed by SONOFF or ITEAD.
